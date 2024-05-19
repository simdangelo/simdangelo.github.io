---
title: "LearningNoteSeries - DataFrame API#3: Join"
date: 2024-05-04
# lastmod: 2022-07-15T15:03:31+02:00
draft: false
summary: Join in Spark.
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

I think that this post will not be so useful for many people. **Join** is something that happens in many other scenarios in the data fields (directly in SQL for data analytics, in Pandas or Polars library for data analysis, etc.), so I'm sure that everyone reading this post is familiar with this topic. However, I decided to write this post just to complete the series on the most common functions found in the Spark DataFrame API.

It is very likely that this will be the last post specifically on the DataFrame API, not because the topics are exhausted, but because it would be pointless to continue to make a list of the functions found in the DataFrame API. In this series of posts we have discussed the most common functions and that is enough. There are dozens of other functions that can be used, but for that there is the Spark documentation that you can consult. This doesn’t mean we will not use DataFrame API anymore. My plan is to address some more specific topics in the future like **Optimization** and **Fine-Tuning** of Spark Jobs and talk about **Internals of Spark**.

# 0. Resources

- “Spark Essentials with Scala” course by Daniel Ciocîrlan (link here: [https://rockthejvm.com/p/spark-essentials](https://rockthejvm.com/p/spark-essentials))

# 1. Start the Project

If you're interested in learning how to create a new Spark project in Scala, refer to the initial blog post on Spark available at the following link: [https://simdangelo.github.io/blog/run-spark-application/](https://simdangelo.github.io/blog/run-spark-application/). In this guide, we utilize the same project that was used in previous tutorials and will continue to use in future ones. You can download this project from the following repository: [https://github.com/simdangelo/apache-spark-blog-tutorial](https://github.com/simdangelo/apache-spark-blog-tutorial).

# 2. Join Definition

**Joins** are used to **combine data from multiple DataFrames** and the underlying principles of this practice are very simple:

- one (or more) column from Table 1 (left) is compared with one (or more) column from Table 2 (right);
- if the condition passes, rows are combined;
- non-matching rows are discarded.

In Spark Joins are **wide transformations**. In order to compute a join, Spark scans the entire DF across the entire cluster, so data is going to be moved around between various Spark nodes. So this involves shuffling, which is **expensive** for performance.

This is why I want to address topics like Optimization in the future: because joins can potentially degrade performance if not well managed.

# 3. Types of Join

Let’s create a new Scala file by creating a new Object (again look at this [https://simdangelo.github.io/blog/run-spark-application/](https://simdangelo.github.io/blog/run-spark-application/)) and we call it `Join`.

Then let’s start with the usual configuration:

```scala
val spark = SparkSession.builder()
    .appName("Spark Joins")
    .config("spark.master", "local[*]")
    .getOrCreate()
spark.sparkContext.setLogLevel("WARN")
```

In addition, add these import declarations to make the entire code we will write today work:

```scala
import org.apache.spark.sql.SparkSession
```

Now let’s create three Spark DataFrames that we’ll use to perform joins:

```scala
import spark.implicits._

val bands = Seq(
  (1, "AC/DC", "Sydney", 1973),
  (0, "Led Zeppelin", "London", 1968),
  (3, "Metallica", "Los Angeles", 1981),
  (4, "The Beatles", "Liverpool", 1960)
)
val bandsDF = bands.toDF("id", "name", "hometown", "year")

val guitars = Seq(
  (0, "EDS-1275", "Gibson", "Electric double-necked"),
  (5, "Stratocaster", "Fender", "Electric"),
  (1, "SG", "Gibson", "Electric"),
  (2, "914", "Taylor", "Acoustic"),
  (3, "M-II", "ESP", "Electric"),
)
val guitarsDF = guitars.toDF("id", "model", "make", "guitarType")

val guitarists = Seq(
  (0, "Jimmy Page", Seq(0), 0),
  (1, "Angus Young", Seq(1), 1),
  (2, "Eric Clapton", Seq(1, 5), 2),
  (3, "Kirk Hammett", Seq(3), 3)
)
val guitaristsDF = guitarists.toDF("id", "name", "guitars", "band")
```

Just to have better in mind these DFs, let’s show them:

![Untitled](images/Untitled.png)

Spark supports these join types: 'inner', 'outer', 'full', 'fullouter', 'full_outer', 'leftouter', 'left', 'left_outer', 'rightouter', 'right', 'right_outer', 'leftsemi', 'left_semi', 'semi', 'leftanti', 'left_anti', 'anti', 'cross'.

Before introduction all types of join, let’s look at this graphical representation that explains better than any words:

![[https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/join-operator?pivots=azuredataexplorer](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/join-operator?pivots=azuredataexplorer)](images/join-kinds.png)

*Source: [https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/join-operator?pivots=azuredataexplorer](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/join-operator?pivots=azuredataexplorer)*

Important note! Consider that I have taken this image because it was not easy to find an image on the web with all the types of joins I want to talk about, but note that Spark does not have joins called rightsemi, rightanti, innerunique.

## 3.1. Left Join, Inner Join, Right Join

Say we want to know to which band each guitarist belongs to, so we join `guitarPlayersDF` with the `bandsDF` based on the band id information. We’ll make left, right, and inner join sequentially:

```scala
// condition
val joinCondition = guitaristsDF.col("band") === bandsDF.col("id")

// left join
val guitaristsBandsDF_left = guitaristsDF.join(bandsDF, joinCondition, "left")
// inner join
val guitaristsBandsDF_inner = guitaristsDF.join(bandsDF, joinCondition, "inner")
// right join
val guitaristsBandsDF_right = guitaristsDF.join(bandsDF, joinCondition, "right")
```

Without going into details, let’s define each join following the image above:

- **Left Join**: combines rows from two tables based on a matching condition, including all rows from the left table and only matching rows from the right table. Any rows from the right table that don't have a match are filled with `NULL` values;
- **Inner Join**: take only the things that match on the left AND the right;
- **Right Join**: take everything on the right + anything on the left that matches;

Here’s the result:

![Untitled](images/Untitled%201.png)

Note that:

- `left` type is equivalent to `left_outer` and `leftouter`;
- `right` type is equivalent to `right_outer` and `rightouter`.

## 3.2. Full-Outer Join

In Full outer join we’ll take everything in the inner join + all the rows in the BOTH table, with `NULL` where the data is missing:

```scala
val guitaristsBandsDF_full_outer = guitaristsDF.join(bandsDF, joinCondition, "full_outer")
```

Here’s the result:

![Untitled](images/Untitled%202.png)

Note that:

- `full_outer` type is equivalent to `outer` and `fullouter` and `full`.

## 3.3. Left-Semi Join

It's like an inner join, but it also cut out the columns from the RIGHT DataFrame:

```scala
val guitaristsBandsDF_left_semi = guitaristsDF.join(bandsDF, joinCondition, "left_semi")
```

Here’s the result:

![Untitled](images/Untitled%203.png)

Note that:

- `left_semi` type is equivalent to `leftsemi` and `semi`.

## 3.4. Left-Anti Join

It keeps the rows in the LEFT DataFrame for which there is no row RIGHT DataFrame satisfying the join condition:

```scala
val guitaristsBandsDF_left_anti = guitaristsDF.join(bandsDF, joinCondition, "left_anti")
```

Here’s the result:

![Untitled](images/Untitled%204.png)

Note that:

- `left_anti` type is equivalent to `leftanti` and `anti`.

# 4. Potential Problems

Let’s consider `guitaristsBandsDF_left` DataFrame and let’s show it again here to better understand the problem:

![Untitled](images/Untitled%205.png)

If we want to select only `id` and `band` (from left table):

```scala
guitaristsBandsDF_left.select("id", "band").show()
```

we’ll got an error because after the join there will be two `id` columns.

Solutions:

1. **rename the column on which we are joining** so that the two keys has now the same name:
    
    ```scala
    val guitaristsBandsDF_left_v2 = guitaristsDF.join(bandsDF.withColumnRenamed("id", "band"), "band", "left")
    ```
    
2. **drop the duplicate column** after the join:
    
    ```scala
    val guitaristsBandsDF_left_v3 = guitaristsDF.join(bandsDF, joinCondition, "left").drop(bandsDF.col("id"))
    ```
    
3. **rename the offending column** before the join:
    
    ```scala
    val bandsModDF = bandsDF.withColumnRenamed("id", "bandId")
    val guitaristsBandsDF_left_v4 = guitaristsDF.join(bandsModDF, guitaristsDF.col("band")===bandsModDF.col("bandId"), "left")
    ```