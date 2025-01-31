# Shuffle Join

In this post let's explore how Spark's shuffle join works and why they are so expensive. 
## What is a shuffle join

Before we dive into shuffle joins, it’s helpful to understand the two main categories of Spark transformations: **narrow** and **wide** transformations.

- **Narrow Transformations**
	These transformations (e.g., `map`, `filter`) operate on each partition’s data **in place**, without requiring data from other partitions. Because no data movement across the cluster is needed, narrow transformations are typically faster and less resource-intensive.
- **Wide Transformations**
	Wide transformations (e.g., `groupBy`, `reduceByKey`, `distinct`, `join`) require data to be **redistributed** (or “shuffled”) across partitions. This means rows in different partitions might need to move to new partitions in order to be processed together. 

A **shuffle join** is a classic example of a wide transformation. It occurs when two datasets (DataFrames or RDDs) need to be joined, but their data is not co-located on the same executor based on the join keys. To resolve this, Spark redistributes (or "shuffles") the data across the nodes, grouping rows with the same join key together on the same executor.

Suppose we performing a shuffle join between two datasets `combined_df` and `lookup_tbl` on the `uuid` column. These two structured dataset are distributed across the cluster in some default partitioning (`spark.sql.files.maxPartitionBytes` default could be `256MB`)

```
result_df = (
	combined_df
	.join(
		lookup_tbl,
		on = "uuid",
		how = "inner"
	)
)
```

Let's breakdown the steps in a shuffle join...
## Step 1: Partitioning the Data (Map)

Partitioning is the process of dividing data into chunks (partitions) that are distributed across nodes in the Spark cluster. In the case of a shuffle join:
- The join key is hashed to determine the partition
- All rows with the same hash value for the join key end up in the same partition.
- This ensures that rows with the same join key from both datasets are co-located for joining.

### How many partitions we will get?

This is determined by the `spark.sql.shuffle.partitions` parameter in your spark session configuration. For many Spark versions (2.x, 3.0 - 3.1) this is set default to 200,which means that each executor will partition their data into 200 partitions (In newer Spark 3.x releases (3.2+), sometimes the default is 200 or `8 * <number_of_cores>`).

Ideally we should have more partitions than available cores to maximize parallelism. If you have fewer partitions than executor cores, some cores will remain idle. If you have too many partitions, the overhead of managing small tasks can degrade performance. For example, suppose `combined_df` file size is 10GB
- `numPartitions = 64`: Each partition is roughly 10GB / 64 = ~156MB, which is manageable for the memory of each executor.
- `numPartitions = 4`: Each partition is 10GB / 4 = 2.5GB, which might exceed the memory of individual executors, leading to out-of-memory errors.

### How do we know where each row should go?

Spark uses a partitioner to map the value of the join key `uuid` to a partition. By default, Spark uses a **hash-based partitioner**. The partition index is determined as:

$$
P_{id}=hash(key) \; mod \; numPartitions 
$$
For example, suppose `numPartitions` is 4 and the `combined_df` contains: 
```
uuid
----
a123
b456
...
c789
e125
```

**Hashing Step:** Each `uuid` value is hashed into an integer using a deterministic hash function:
```
hash("a123") = 42
hash("b456") = 15
hash("c789") = 98
hash("d192") = 63
```
Each unique key will result in an unique hash integer. 

**Modulo Operation** produce the partition index for each key:
```
nunmPartitions = 4
partition("a123") = 42 % nunmPartitions = 2
partition("b456") = 15 % nunmPartitions = 3
partition("c789") = 98 % nunmPartitions = 2
partition("d192") = 63 % nunmPartitions = 3
```

**Partition Mapping**:
- Partition 2: Rows with `uuid` `a123` and `c789`
- Partition 3: Rows with `uuid` `b456` and `d012`

### Writing intermediate results

Spark executors will write these partitions as intermediate files to disk. 

You can consider the Step #1 partitioning stage as the **Map Phase**, where the goal is to repartition the original `combined_df` and `lookup_tbl` partitions within each executor into new partitions. The driver node will create map tasks where:
- Each map task processes its assigned input partition
- Rows are divided into shuffle partitions based on the partitioning logic (e.g. `hash(key) % numPartitions`)
- Each shuffle partition is written to the **local disk** of the executor, organized as intermediate shuffle file. 


## Step 2: Shuffling (Reduce)

The goal at this step is to combined shuffle partitions that shared the same partition index across executors. 

### Does Data Shuffling Happen Between Executors or Nodes?

Let's review the definitions: 
- **Executors** are JVM processes running on worker nodes in the Spark cluster. They handle the execution of tasks and manage data storage (e.g., in memory or on disk). 
- **Nodes** are physical or virtual machines in the cluster, and each node may have multiple executors running on it depending on your Spark configuration.

**Data shuffling happens between executors**, regardless of whether they are on the same or different nodes. If two executors (handling different partitions) reside on the same node, the data is transferred locally. Otherwise, the data is sent across the network. This is why shuffling can be really expensive! 

- **Same Node:** Local data transfer is much faster because it avoids network overhead. It relies on the node’s I/O speed and disk performance. 
- **Different Nodes:** When executors on different nodes exchange shuffle data, it must be transferred over the network. Spark uses **HTTP-based communication** (or Netty) to move the data between nodes, which introduces **network latency** and increases job execution time.

During the **reduce phase**, each reduce task is responsible for one shuffle partition. The reduce task collects the corresponding shuffle data that shared the same index from all map tasks across all executors. For example, if partition 42 is assigned to a reduce task, that task fetches all data for partition 42 from all executors.

### How Does Spark Know the Location of Shuffle Data?

The **Driver** orchestrates the shuffle by keeping track of:
- Which executors are processing which partitions.
- The mapping of shuffle data to executors and partitions.

Each executor has a **Block Manager** that:
- Map phase: registers the locations of the shuffle files it has written with the driver.
- Reduce phase: handles requests for shuffle data from other executors.

So when a reduce task needs data for a specific shuffle partition, it queries the **Driver** to get the location(s) of the shuffle files for that partition. It then fetches the shuffle data directly from the corresponding executors.

Note that the partitioning and shuffling happened to both `combined_df` and `lookup_tbl`. 

## Step 3: Joining

Finally, each partition of the join key from `combined_df` is co-located with the corresponding partition of the same key `lookup_tbl`. It is time to join the shuffle partitions together. 


### Different Join Strategies

Spark supports multiple join types, each suited for specific scenarios based on dataset size, data distribution, and memory constraints. Here’s a breakdown of join types: 


| **Join Type**           | **Description**                                                                                  | **When to Use**                                                                                       |
| ----------------------- | ------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| **Broadcast Hash Join** | Broadcasts the smaller dataset to all executors and uses a hash table for joining.               | When one dataset is small enough to fit in executor memory (e.g., < 10MB-100MB depending on cluster). |
| **Shuffle Hash Join**   | Both datasets are shuffled and partitioned by join key. A hash table is built for one partition. | When datasets are moderately large, and memory is sufficient to hold one partition in memory.         |
| **Sort-Merge Join**     | Sorts both datasets on the join key and merges them in sorted order.                             | When both datasets are large, and there’s insufficient memory for hash joins.                         |
| **Cartesian Join**      | Performs a cross-product of the two datasets (all combinations).                                 | When no join condition is specified (should be avoided unless explicitly required).                   |
| **Skew Join**           | Handles skewed join keys by redistributing skewed keys more evenly across partitions.            | When datasets are large and skewed (used in conjunction with AQE or salting).                         |

When no explicit join type is specified, Spark uses Sort-Merge Join. This is because it’s robust and works well for large datasets without requiring one dataset to fit in memory. However, if broadcast join conditions are met (e.g., one dataset is smaller than the `spark.sql.autoBroadcastJoinThreshold`), Spark uses Broadcast Hash Join. 

We won't into details on how each join strategy is implemented. You can learn more about the implementations at this [Introduction to Database Systems](https://www.youtube.com/watch?v=1qZeNZvEgGg&list=PLzzVuDSjP25RQb_VhEBFWFiB7oS9APM7h) course offered by UC Berkeley. Here's a general guide for you to select a join method based on the dataset size:

|**Dataset A Size**|**Dataset B Size**|**Recommended Join**|**Reason**|
|---|---|---|---|
|Small|Small|**Broadcast Hash Join**|No shuffle required; small datasets are broadcast to all nodes.|
|Small|Large|**Broadcast Hash Join**|Broadcast the small dataset to avoid shuffling the large dataset.|
|Moderate|Moderate|**Shuffle Hash Join** or **Sort-Merge Join**|Depends on memory availability. Use Shuffle Hash Join if memory is sufficient.|
|Large|Large|**Sort-Merge Join**|Handles large datasets efficiently by sorting and merging.|
|Large + Skewed|Any|**Skew Join**|Redistributes skewed keys to balance partition sizes and avoid task stragglers.|

### Adaptive Query Execution (AQE) 

Tired of manual tuning? Adaptive Query Execution (AQE) is a feature in Spark that dynamically adjusts query plans at runtime based on the actual data. It has been enabled by default since Spark `3.2.0`. For joins, AQE can:

- Dynamically select efficient join strategies
- Handle data skew to balance the workflow, preventing out-of-memory errors
- Dynamically coalesce shuffle partitions

To explore more how AQE can enhance performance tuning, you can refer to Spark's [official documentation](https://spark.apache.org/docs/3.5.3/sql-performance-tuning.html#adaptive-query-execution) for detailed information.

## Why are shuffle joins expensive?

Now we understand how shuffle join works, let's summarize why are they so expensive.
### Disk I/O Overhead

During the “map” phase, each executor writes intermediate shuffle files to disk (one file per partition). In the “reduce” phase, other executors read these files from disk.

Frequent writes and reads to disk add latency and can create performance bottlenecks.

### Network I/O Overhead

If the required shuffle data is located on different nodes, executors must transfer data across the network. Network transfers can be slow compared to local disk reads, especially in large clusters where data must travel multiple hops.

### Serialization/Deserialization Cost

Data moving between executors or being written to disk must be serialized and deserialized. This process consumes CPU time, especially when dealing with large records or complex data types.

### Large Number of Shuffle Files

Every partition of every task writes out a shuffle file for each reduce partition. In large jobs, this can result in thousands (or even millions) of small files, adding file management overhead for both Spark and the underlying filesystem.

Also note that each shuffle partition requires a new task to handle the reduce side. This can inflate the total number of tasks and increase scheduling overhead.

### Data Skew

If certain join keys are much more common than others (data skew), some partitions become disproportionately large, slowing down the entire stage. 

In this example we joined `combined_df` and `lookup_tbl` on the key `uuid`. The unique identifier is distinct and uniform which prevented data skew during the hashing phase. However, if the join key is a categorical variable then the distribution of the values might not be uniform hence lead to data skew. 
### Memory Pressure

The executors doing the reduce must read all corresponding shuffle files for their partition, which can stress memory if the partition is large. Insufficient memory can lead to costly spills to disk or even job failures.

