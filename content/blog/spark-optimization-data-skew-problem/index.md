---
title: "LearningNoteSeries - SparkOptimization#2: How to solve Data Skew problem"
date: 2024-06-08
# lastmod: 2022-07-15T15:03:31+02:00
draft: false
summary: Data Skew in Spark.
# categories: [curated lists]
tags: [coding, spark, tutorial]
# ShowToc: true
# weight: 2
# cover:
#   image: "images/burns-mystery-box.jpg"
#   alt: "burns-mystery-box.jpg"
#   caption: "Now, you can either have the washer and dryer where the lovely Smithers is standing, or you can trade it all in for what's in this box."
#   relative: true
---

# 1. Data Skew

## 1.1. Data Skew explained

**When does Data Skew happens?**

Let’s assume a scenario where you have data split in **5 partition** (p3 is the largest one, p2 is the smallest one) and an hardware configuration consisting in **1 executors** with **5 cores**:

![Untitled Diagram-Page-6.drawio.svg](images/Untitled_Diagram-Page-6.drawio.svg)

Let’s see how these partitions are processed inside the executor:

![Untitled Diagram-Page-7.drawio.svg](images/Untitled_Diagram-Page-7.drawio.svg)

From the dotted line on, only one core is working, while all the others are sitting **IDLE**. This means that there is an **uneven utilisation of resources** and you’re paying for resources you’re not even using. The ideal scenario should be the following one:

![Untitled Diagram-Page-7.drawio (2).svg](images/Untitled_Diagram-Page-7.drawio_(2).svg)

Let’s try to give a definition to **Data Skew**: In Apache Spark, the Data Skew problem occurs when the distribution of data across the partitions is uneven. This imbalance can lead to some partitions being much larger than others, causing certain tasks to take significantly longer to process. As a result, the overall job execution time increases, leading to inefficiencies and reduced performance. Data skew typically arises from the presence of highly frequent keys in operations such as joins, groupBy, and aggregations.

**What kind of operations can cause Data Skew?**

- `groupBy()` operation.
    
    Suppose an operation like:
    
    ```scala
    df.groupBy("country").count()
    ```
    
    If in your dataset there is a `country` value which is more numerous compared to the others, it turns out that whichever core getting process the partition where that country is present, is going to take a lot of more time compared to the other partitions:
    
    ![Untitled Diagram-Page-7.drawio (5).svg](images/Untitled_Diagram-Page-7.drawio_(5).svg)
    
- `join()` operation.
    
    The reason is the same as grouping operation: there is a massive presence of some specific value of `product_id` that causes `join skew`.
    

**Negative Consequences of Data Skew**

- Job is taking time
- uneven utilization of resources (cores are sitting IDLE as we have seen before)
- Out Of Memory Errors

## 1.2. Data Skew example

Let’s start by creating a new Scala file called DataSkew and let’s write the usual configuration:

```scala
val spark = SparkSession.builder
  .appName("Data Skew Example")
  .master("local[4]")
  .config("spark.sql.adaptive.enabled", "false")
  .getOrCreate()
  
spark.sparkContext.setLogLevel("WARN")
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
```

Two important things to note:

- **Adaptive Query Execution (AQE)** is one of the possible solution to Data Skew problem, so I disabled that in order to show the problem.
- Broadcast Join is another possible solution to Data Skew problem, so again I disable that.

For this example I used the data that Afaque Ahmad shared in his GitHub Repo (link above). Let’s import a skewed DF, where a specific customer has insanely more transaction than the others, and another DF we’ll join with:

```scala
val transactions_file = "src/main/resources/data/data_skew/transactions.parquet"
val customer_file = "src/main/resources/data/data_skew/customers.parquet"

val df_transactions = spark.read.parquet(transactions_file)
val df_customers = spark.read.parquet(customer_file)
```

Let’s show the skewness in the `df_transactions` DF:

```scala
val df_transactions_grouped = df_transactions
  .groupBy("cust_id")
  .agg(countDistinct("txn_id").alias("ct"))
  .orderBy(desc("ct"))
```

![Untitled](images/Untitled.png)

This means that, if you join this DF with someone else on `cust_id` as join key, this join is going to be skewed at that `cust_id`. So, let’s take a join:

```scala
val df_txn_details = df_transactions.join(df_customers, "cust_id", "inner")
```

Let’s take an action on `df_txn_details` in order to start the computation and, in order to keep the SparkUI alive, let’s put as following in the `main` function:

```scala
df_txn_details.count()
Thread.sleep(1000000)
```

Now, let’s go in the SparkUI where we can look at the Data Skew problem visually:

![Untitled](images/Untitled%201.png)

Let’s click on the JobId 2 corresponding to the `.count()` action:

![Untitled](images/Untitled%202.png)

Now let’s explore the details of StageId 4, that is the one where the join is taken place:

![Untitled](images/Untitled%203.png)

You can also look at the Summary Metrics table where it’s clear that the Median Duration of a Task is 34ms, but the max is 4s:

![Untitled](images/Untitled%204.png)

From these pictures, we can clearly observe the **Data Skew problem**, where parallelism is severely compromised: only one core is active while the others remain IDLE, resulting in parallelism dropping to 1 after a certain point. In contrast, the ideal scenario is where all cores work simultaneously, utilizing parallelism to its fullest extent.

# 2. Solutions

## Solution 1: **Adaptive Query Execution (AQE)**

**Adaptive Query Execution (AQE)** is an optimization technique in Spark SQL that makes use of the runtime statistics to choose the most efficient query execution plan, which is enabled by default since Apache Spark 3.2.0. Statistics can be:

- \# of bytes read (basically the size of your input dataset)
- \# of partitions

The kind of tuning that AQE gives us is among the following three:

- **Coalescing Post Shuffle Partitions**. Let’s say you have 200 shuffle partition by default and you have a dataset you want to join on a specific join key and this join key has only 15 distinct values. This means that the remaining remaining 185 partitions are going to be empty. What Spark does is that is going to coalesce all of the partitions into 15 partitions, so that it reduces the number of partitions.
- **Optimizing Joins**. Spark converts `SortMergeJoin` (which involves shuffles) into a Broadcast Join `BroadcastHashJoin` (which does not involve shuffle).
- **Optimizing Skew Joins**. Spark splits the skewed partition into smaller partitions. It’s written in the documentation:
    
    ![Untitled](images/Untitled%205.png)
    
    To make this optimization technique effective both of the following configuration must be enabled:
    
    ```scala
    spark.sql.adaptive.enabled
    spark.sql.adaptive.skewJoin
    ```
    

### **Code Example**

Let’s enable AQE by inserting in the code:

```scala
spark.conf.set("spark.sql.adaptive.enabled", "true") // activate AQE
spark.conf.set("spark.sql.adaptive.skewedJoin.enabled", "true") // activate AQE
```

Then let’s perform again the same join operation as before and look a again at the SparkUI:

![Untitled](images/Untitled%206.png)

If you look at the “Tasks” column you can already notice that the total number of stages is much less than before. Let’s look at the details of the JobId 4:

![Untitled](images/Untitled%207.png)

Now let’s explore the details of StageId 6, in particular the Event Timeline and the Summary Statistics:

![Untitled](images/Untitled%208.png)

Compared to the previous case, the situation is significantly different.

- there are only 4 stages now, compared to the 200 stages before;
- the four stages are executed in roughly the same amount of time, or at least within the same order of magnitude (3 seconds for the shortest, 6 seconds for the longest).

In the SparkUI let’s click on the **SQL/DataFrame** tab to see the query plan and let’s compare the plan without and with AQE.

Without AQE:

![Untitled.png](images/Untitled%209.png)

With AQE:

![Untitled.png](images/Untitled%2010.png)

Note that the two plans are very similar, except for an additional step called `AQEShuffleRead`, which is introduced when AQE is enabled for each DataFrame performing the join. This is why, with AQE, we have seen only 4 stages instead of 200 in the Event Timeline sections: AQE performed a coalesce operation, reducing the total number of partitions of each DataFrame from **200 to 4**. You can verify this by checking the “**number of partitions”** statistics in these blocks.

Notice that AQE may not be always the best option.

## Solution 2: Broadcast Join

`SortMergeJoin` is the classical join operation and it assumes that both branches of this joins need to be shuffled and sorted by the column that’s being joined on. So, it’s clear that it involves many operations. Broadcast join, on the other hand,  is considered to be better than `SortMergeJoin` for certain cases, where one of the table is smaller as compared to the other table.

### `SortMergeJoin`

As the name itself suggests this operation involves many different steps:

- shuffling (most expensive operation)
- sorting
- merging

**Shuffling**

Let’s suppose you have 2 dataset (`Transactions` and `Customers`) with 3 partitions each and you are doing a join on `cust_id` and `id` columns, respectively. In the `Transactions` the attribute `cust_id` is abnormally distributed because there’s a specific `cust_id` that made a lot of transactions.

![Untitled Diagram-Page-1.drawio.svg](images/Untitled_Diagram-Page-1.drawio.svg)

As we already know, to join datasets, the join keys from both tables must be on the same executors, which requires shuffling. Generally when Spark operation performs data shuffling the resulting dataset increases the partition number to **200 by default**. For simplicity, let’s assume that the number of shuffle partitions (that you can set with the SQL configuration `spark.sql.shuffle.partitions`) is 3. So shuffling means transfer of data over the network and this means that values of `cust_id` and `id` would end up in different partition.

Which is the logic behind a key going into a specific partition? Essentially, how does Spark determine which rows should end up on which executors? By default, Spark uses **Hash Partitioning**, meaning that the for each key, Spark computes the Hash Code and then applies a modulo operation with the number of partitions. This determines the partition number where the key-value pair should be stored. The formula is:

`partition = hash(key1, key2, ..., key_n) % number_of_shuffle_partitions`

Only for calculation simplicity, we’ll use:

`partition = key % number_of_shuffle_partitions`

Let’s compute the `partition` value only for data in Executor 1 (but it’s the same for the other ones) by applying the formula above:

![Untitled Diagram-Page-2.drawio.svg](images/Untitled_Diagram-Page-2.drawio.svg)

**Sorting**

After rearranging the rows into new partitions thanks to Hash Partitioning, the join keys in both tables are sorted:

![Untitled Diagram-Page-3.drawio (1).svg](images/Untitled_Diagram-Page-3.drawio_(1).svg)

**Merging**

Finally the matching between the two tables in every executor is executed and the `SortMergeJoin` is done.

The final result will be:

![Untitled Diagram-Page-4.drawio.svg](images/Untitled_Diagram-Page-4.drawio.svg)

### Broadcast Join

As we already said in the last post on this blog, in the broadcast join the smaller table will be copied into every executors:

![Untitled Diagram-Page-5.drawio.svg](images/Untitled_Diagram-Page-5.drawio.svg)

So, why Broadcast joins can help to solve **Data Skew** problem? It’s because Broadcast joins are immune to skewed input dataset and the reason is that you have a complete flexibility to partition in the way you want, so there is no compulsion that you have to partition it by the join key. So what you can do is to perform a `repartition(3)` to **evenly repartition the big table in each executor** and then perform a **broadcast join**. In this case the three executors will work on the same amount of data each. 

In our code:

```scala
val df_txn_details_broadcasted = df_transactions.join(broadcast(df_customers), "cust_id", "inner")
```

3.2 seconds vs 10 seconds of the `SortMergeJoin.`

Notice the **limitations of broadcast joins**: the broadcasted dataset needs to fit in the driver & executor memory, and if you have many executors, it may take longer than shuffle merge, it could in fact timeout.

Let’s try our code first by disabling AQE:

```scala
spark.conf.set("spark.sql.adaptive.enabled", "false") // disable AQE
spark.conf.set("spark.sql.adaptive.skewedJoin.enabled", "false") // disable AQE

val df_txn_details_broadcasted = df_transactions.join(broadcast(df_customers), "cust_id", "inner")
```

This is the result:

![Untitled](images/Untitled%2011.png)

Then let’s enabling AQE:

```scala
spark.conf.set("spark.sql.adaptive.enabled", "true") // activate AQE
spark.conf.set("spark.sql.adaptive.skewedJoin.enabled", "true") // activate AQE

val df_txn_details_broadcasted = df_transactions.join(broadcast(df_customers), "cust_id", "inner")
```

Here’s the result:

![Untitled](images/Untitled%2012.png)

In this particular scenario, the **Broadcast Join** performs effectively both with AQE and without it. Indeed, it's worth noting that the job duration is significantly slower compared to previous cases.

## Solution 3: Salting

To **Salting** basically refers to adding randomness to a key in order to be able to evenly distribute it.

Suppose you have a **Table1** (bigger one, the skewed dataset) and **Table2** (smallest one) that you want to join on `id` column. This column on Table1 has 1 million rows with value 1, 5 rows with value 2, and 10 rows with value 3. It’s of course very skewed. If we were to perform a classical `SortMergeJoin` and we set the number of shuffle partition to 3, it happens that in the first partition there would be 1 million elements, in the second partition 5 elements, and in the third partition 10 elements because of the Hash Partitioning formula `partition=hash(value) % number_of_shuffle_partitions` (we’ll use a simplified version of that formula):

- `1%3=1`: `1` (count=1.000.000) goes into partition 1
- `2%3=2`: `2` (count=5) goes into partition 2
- `3%3=0`: `3` (count=10) goes into partition 0

![Untitled Diagram-Page-8.drawio (4).svg](images/Untitled_Diagram-Page-8.drawio_(4).svg)

This third solution to Data Skew problem is called **Salt Technique**.

### Step 1: **Choose a salt number**

It indicates how much we want to distribute the data. In our case let’s choose a number between 0 included and 3 not included `[0, 3)` and let’s add this new column to each of the rows:

![Untitled Diagram-Page-9.drawio (2).svg](images/Untitled_Diagram-Page-9.drawio_(2).svg)

Now, instead of doing the join on `id` columns, we’ll do the join on `(id, salt)`. Since this becomes the new join key, the Hash Partitioning formula that decides which row is going to which partition becomes:

`hash(id, salt) % 3`

The advantage of this is that, while without salting we had `hash(1) % 3 = N` and so N remained the same for every rows with `id` equal to `1` (implying that every rows with `id==1` go to the same partition), now we’ll have 3 different hashes for `id==1`:

`partition = hash(1, 0) % 3 = 0`

`partition = hash(1, 1) % 3 = 1`

`partition = hash(1, 2) % 3 = 2`

So now we’re able to distribute `1`s `id` values more effectively into 3 different partitions that appear more balanced with a distribution similar to the following one:

![Untitled Diagram-Page-8.drawio (5).svg](images/Untitled_Diagram-Page-8.drawio_(5).svg)

Remember the key point of this step: we have just **created a new join key** that it’s no longer composed by `id` column, but it’s now composed of `(id, salt)`. This implies that the Hash formula is no longer computed on `id` only, but on `(id, salt)`.

### Step 2: **Add Salt List for each key of Table2**:

Since we have just created a new join key composed of `(id, salt)`, we now need a way to create this new join key in Table2 as well.

So, take Table2 and append all the salt values to the `id` column as a list:

![Untitled Diagram-Page-9.drawio (7).svg](images/Untitled_Diagram-Page-9.drawio_(7).svg)

### Step 3: **`explode` Salt List to get Salt for each Key**

The Spark `explode` function generates a new row for each element in an array. In this context, we will use the `explode` function on the `salt_list` column, resulting in three new rows for each `id` value, each corresponding to one of the values contained in the `salt_list`. Let’s see it graphically:

![Untitled Diagram-Page-9.drawio (8).svg](images/Untitled_Diagram-Page-9.drawio_(8).svg)

The result is that for Table2, we now have a new join key composed of the `(id, salt)` columns, making it ready for the join operation with Table1.

### Step 4: Make **the Join**

Now, you can easily perform the join operation without encountering skewed data. As shown in the following image, rows with `id==1` are no longer confined to the same partition; instead, they are distributed across all partitions. I want repeat the reason again: that’s because the Hash formula is no longer computed on `id` only, but on `(id, salt)`.

![Untitled Diagram-Page-10.drawio.svg](images/Untitled_Diagram-Page-10.drawio.svg)

### **Code Example**

```scala
spark.conf.set("spark.sql.shuffle.partitions", "3")
spark.conf.set("spark.sql.adaptive.enabled", "false")

// Create df_uniform DataFrame
val df_uniform = spark.createDataFrame((0 until 1000000).map(Tuple1(_))).toDF("value")

// Create skewed DataFrames
val df0 = spark.createDataFrame(Seq.fill(999990)(0).map(Tuple1(_))).toDF("value").repartition(1)
val df1 = spark.createDataFrame(Seq.fill(15)(1).map(Tuple1(_))).toDF("value").repartition(1)
val df2 = spark.createDataFrame(Seq.fill(10)(2).map(Tuple1(_))).toDF("value").repartition(1)
val df3 = spark.createDataFrame(Seq.fill(5)(3).map(Tuple1(_))).toDF("value").repartition(1)

// Union skewed DataFrames
val df_skew = df0.union(df1).union(df2).union(df3)

// Add partition column and show partition counts
val df_skew_show = df_skew
  .withColumn("partition", spark_partition_id())
  .groupBy("partition")
  .count()
  .orderBy("partition")

print("Data distribution across Partitions before Salting Technique:\n")
df_skew_show.show()

val SALT_NUMBER = spark.conf.get("spark.sql.shuffle.partitions").toInt

// Add salt column to df_skew
val df_skew_with_salt = df_skew.withColumn("salt", (rand() * SALT_NUMBER).cast(IntegerType))

// Add salt_values and explode to create salt column in df_uniform
val saltValues = (0 until SALT_NUMBER).map(lit(_)).toArray
val df_uniform_with_salt = df_uniform
  .withColumn("salt_values", array(saltValues: _*))
  .withColumn("salt", explode(col("salt_values")))

// Perform the join operation between df_skew_with_salt and df_uniform_with_salt
val df_joined = df_skew_with_salt.join(df_uniform_with_salt, Seq("value", "salt"), "inner")

// Add partition column and show partition counts
val df_joined_show = df_joined
  .withColumn("partition", spark_partition_id())
  .groupBy("partition")
  .count()
  .orderBy("partition")

print("Data distribution across Partitions after Salting Technique:\n")
df_joined_show.show()
```

Here’s the result:

![Untitled](images/Untitled%2013.png)

See how the data is now distributed evenly across all partitions.