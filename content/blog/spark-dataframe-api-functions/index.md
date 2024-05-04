---
title: "#8 DataFrame API#2: Columns, Expressions, and Functions"
date: 2024-04-13
# lastmod: 2022-07-15T15:03:31+02:00
draft: false
summary: Most commonly used functions in Spark.
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

# 0. Resources:

- “Spark Essentials with Scala” course by Daniel Ciocîrlan (link here: [https://rockthejvm.com/p/spark-essentials](https://rockthejvm.com/p/spark-essentials))
- https://sparkbyexamples.com/


# 1. Start the Project

If you're interested in learning how to create a new Spark project in Scala, refer to the initial blog post on Spark available at the following link: [https://simdangelo.github.io/blog/run-spark-application/](https://simdangelo.github.io/blog/run-spark-application/). In this guide, we utilize the same project that was used in previous tutorials and will continue to use in future ones. You can download this project from the following repository: [https://github.com/simdangelo/apache-spark-blog-tutorial](https://github.com/simdangelo/apache-spark-blog-tutorial).

---

# 2. Basic functions

## 2.1. Select columns

**Selecting columns of a DataFrame** is perhaps the most used operation is Spark.

The two functions that are used to do that are: `.select()` and `selectExpr()`. As you’ll see there are no practical reasons to choose one instead of the other, so feel free to use the one that you prefer. Whatever you choose, I suggest you to stay consistent with your choice in all your code to improve the readability. Just to say, `.select()` is by far the most used one.

Before diving into the syntax of each function, let’s describe briefly what these functions do internally and then we’ll represent them graphically. Remember that Spark DataFrames are split in partitions in between nodes and the cluster so, when I use a select function to select any number of columns from a DF, those columns are being selected on every partition on every node where the DF resides. After the selections, you will obtain a new DF with those columns and that will be reflected on every node in the cluster. As I described in one of the last posts in this blog, these functions represent a Transformation, specifically a **Narrow Transformation:**

- **Transformation** because it is an operation applied to an RDD that results in the creation of a new RDD and Spark does not immediately compute the result until an **Action** is called.
- **Narrow** because every partition in the original DF has exactly one corresponding output partition in the resulting DF.

Let’s represent it graphically:

![Untitled](images/Untitled.png)

*Source: “Spark Essentials with Scala” course by Daniel Ciocîrlan (link here: https://rockthejvm.com/p/spark-essentials)*


Note: The technical term for a selection is **Projection**. So we are projecting the DF into a new DF. Projection is a term used from the theory of databases.

Let’s download a `csv` file that we’ll use as example in this post: [https://www.kaggle.com/datasets/kanchana1990/trending-ebay-watch-listings-2024](https://www.kaggle.com/datasets/kanchana1990/trending-ebay-watch-listings-2024) and put it in the `src/main/resources/data` folder in our Scala project.

Let’s start with the usual configuration (read my other posts to know to do that) and then read the DF:

```scala
val spark = SparkSession.builder()
    .appName("Columns, Expressions, and Functions")
    .config("spark.master", "local[*]")
    .getOrCreate()

// read a DataFrame
val df: DataFrame = spark.read
  .option("header", "true")
  .option("delimiter", ",")
  .option("inferSchema", "true")
  .csv("src/main/resources/data/ebay_watches.csv")
```

In addition, add these import declarations to make the following code work:

```scala
import org.apache.spark.sql.{DataFrame, SparkSession}
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._
import org.apache.spark.sql.expressions.Window
```

### `.select()`

There are several ways to select columns with this function:

```scala
// select() function to select columns
import org.apache.spark.sql.functions.{col, expr}
import spark.implicits._

val select1 = df.select("itemLocation","lastUpdated", "sold")
val select2 = df.select(expr("itemLocation"), expr("lastUpdated"), expr("sold"))
val select3 = df.select(col("itemLocation"), col("lastUpdated"), col("sold"))
val select4 = df.select($"itemLocation", $"lastUpdated", $"sold")
val select5 = df.select(df("itemLocation"), df("lastUpdated"), df("sold"))
```

If we print apply `.show()` function to all these functions, we’ll see that all these expressions return the same result, that is:

![Untitled](images/Untitled%201.png)

Note that:

- the first two options are the most used ones;
- to use `$""` expression you need to import `spark.implicits._`;
- to use `col` and `.expr` you need to import `org.apache.spark.sql.functions.{col, expr}`;
- in `.expr()` method you can pass a **SQL-like string** and expressions are a powerful constructs and they will allow you to process DF in almost any fashion that you like.

Let’s make an example to understand the power of `.expr()`.

```scala
val df2 = df.select(
  col("itemLocation"),
  col("sold"),
  expr("sold * 2 as double_sold"),
  expr("CASE WHEN itemLocation IS NULL THEN 'null_value' ELSE itemLocation END as itemLocation_mod")
)
```

Let’s apply `.show()` and here’s the result:

![Untitled](images/Untitled%202.png)

Technically, `.select()` function can take `String` (corresponding to `select1` case) or `Column` (corresponding to `select2`, `select3`, `select4` cases) types as arguments. It is worth mentioning because this aspect differs from `.selectExpr()` function.

### `.selectExpr()`

This function is used for selecting and transforming columns using **SQL expressions**. Besides selecting columns, it allows you to use SQL expressions to manipulate columns (without creating a temporary table and views) including arithmetic operations, string manipulations, and aggregation functions.

This function doesn’t have a signature to take `Column` type, but it takes only `String`.

You can consider this function as a more powerful `.expr()` function. Let’s take the expression to create df2 in the previous paragraph and let’s rewrite it with `.selectExpr()` function:

```scala
val df3 = df.selectExpr(
  "itemLocation",
  "sold",
  "sold * 2 as double_sold",
  "CASE WHEN itemLocation IS NULL THEN 'null_value' ELSE itemLocation END as itemLocation_mod"
)
```

This expression for `df3` returns the exact same result as `df2`.

## 2.2. Create new columns `.withColumn()`

Actually, we have already seen how to extends a DF with an additional column and create a new column with `.select()` function. Indeed, besides selecting existing column, this function allows to create new column.

A more explicit function to create new columns is `withColumn()`. Let’s use both functions and we’ll see that the result is exactly identical:

```scala
val df4 = df
  .select(
    col("sold"),
    (col("sold") * 2).alias("double_sold_v1") // create column version 1
  )
  .withColumn("double_sold_v2", col("sold")*2) // create column version 2
```

Here’s the result:

![Untitled](images/Untitled%203.png)

## 2.3. Evaluate conditions `.when()`

This is one of the most utilised function in Spark. `when()` function is a conditional expression that evaluates a set of conditions and returns a corresponding value based on the first condition that evaluates to true. This condition must be followed by the `.otherwise()` function, which takes into account all those records that do not match the condition in the `.when()` function.

Let’s say we want to fix the column `seller` because a specific seller has changed its name:

```scala
val df5 = df
  .withColumn("seller_new",
    when(col("seller")==="Direct Luxury", lit("Luxury Boutique"))
    .otherwise(col("seller")))
```

**Be careful**: don’t forget the `.otherwise()` function, otherwise all records that don’t match the condition are set to `NULL`.

Here’s the result:

![Untitled](images/Untitled%204.png)

You can also chain more than one conditions sequentially:

```scala
val df5_2 = df
  .withColumn("seller_new",
    when(col("seller") === "Direct Luxury", lit("Luxury Boutique"))
      .when(col("seller") === "WATCHGOOROO", lit("Watch Gooroo"))
      .when(col("seller") === "ioomobile", lit("Ioo Mobile"))
      .otherwise(col("seller")))
```

## 2.4. Rename columns `.withColumnRenamed()`

The function `.withColumnRenamed()` takes two parameters:

- the first one is the existing name
- the second one is the new name

Actually we have already seen in the previous chapter a function to rename a column, that is `.alias()`. Even in this case both functions return the same result:

```scala
val df6 = df.withColumnRenamed("itemLocation", "item_location")
val df6_2 = df.select(col("itemLocation").alias("item_location"))
```

Of course in `df6` all columns are selected and one is renamed, while in `df6_2` only one column is selected and that one is renamed.

## 2.5. Remove columns `.drop()`

Remove a column with `drop`:

```scala
val df7 = df.drop("availableText", "item_location")
```

## 2.6. Filter records `.filter()`

There are several ways to filter a DF with `.filter()` function (or equivalently with `.where()`):

```scala
val df8 = df.filter(col("seller")==="Direct Luxury")
val df8_2 = df.where(col("seller")=!="Direct Luxury")
val df8_3 = df.filter("seller = 'Direct Luxury'")
```

Note that you need to use `===` and `=!=` for column expressions because `==` and `!=` are standard Scala Operators. You can of course whichever logical and arithmetical operators you want.

You can also extend the potentiality of filtering by using `.isin()` (if you want to include more than one element) and/or `.not()` (corresponding to `~` in PySpark) to deny a statement. For instance:

```scala
val df8_4 = df.filter(col("seller").isin("Direct Luxury", "Sigmatime", "ioomobile"))
val df8_5 = df.filter(not(col("seller").isin("Direct Luxury", "Sigmatime", "ioomobile")))
```

You can also filter all records that have a specific column with `NULL` values with `.isNull` (or the opposite with `.isNotNull`):

```scala
val dfWithNull = df.filter(col("itemLocation").isNull)
val dfWithoutNull = df.filter(col("itemLocation").isNotNull)
```

The result of `df8` is:

![Untitled](images/Untitled%205.png)

You can also chain filters:

```scala
// chain filters
val df9 = df.filter(col("seller")==="Direct Luxury").filter(col("sold")>1)
val df9_2 = df.filter((col("seller")==="Direct Luxury") and (col("sold")>1))
val df9_3 = df.filter((col("seller") === "Direct Luxury").and(col("sold") > 1))
val df9_4 = df.filter("seller = 'Direct Luxury' and sold > 1")
```

You can also use or keyword, or symbols (`&&` to use a boolean AND and `||` to use a boolean OR)

Note that `df9_3` uses `and` keyword without parenthesis because the `and` method is **infix** and this looks like more natural language.

## 2.7. Order DataFrame rows `.orderBy()`

`.orderBy()` function is pretty simple: it returns a new DF with rows ordered by one or more columns. By default the order is ascending:

```scala
val orderBy = df
  .orderBy(col("sold"), col("title"))
```

Here’s the result:

![Untitled](images/Untitled%206.png)

**Be careful**:

- make sure that columns are casted with the right data type. If `sold` column is string format, the order was alphabetically, while if it is numeric format the order is from the smallest to the biggest number;
- since `sold` column contains `NULL` value, they appear first. If you want to keep the same order, but want `NULL`s to appear last, you can use `asc_nulls_last` function:
    
    ```scala
    val orderBy = df
      .orderBy(asc_nulls_last("sold"), col("title"))
    ```
    
    Now the result is:
    
    ![Untitled](images/Untitled%207.png)
    
- if you want to order in descending order you can use `.desc(*col*)` and `.desc_nulls_last(*col*)` if you want to deal with `NULL`:
    
    ```scala
    val orderBy_desc = df
      .orderBy(desc_nulls_last("sold"), desc("title"))
    ```
    

## 2.8. Summary statistics `.min()`, `.max()`, `.avg()`

The usage of these functions is pretty trivial, so let’s make an example right now:

```scala
val df_summary = df.select(min("sold"), max("sold"), avg("sold"))
```

Here’s the result:

![Untitled](images/Untitled%208.png)

**Be careful**: these statistic summary functions must be applied only on `int`, `long`, `double`, and all the others numeric data types! So, before using them check if the schema is well applied. For instance, if `sold` had been a `string`, I could have circumvented the problem and used these functions by forcing the column to an `integer` (or other numeric data types ) with the `.cast()` function:

```scala
val df_summary_casting = df.select(min(col("sold").cast("int")), max(col("sold").cast("int")), avg(col("sold").cast("int")))
```

## 2.9. Union more DataFrames

### `.union()`

To union two or more DFs with same number of columns you can use `.union()` function:

```scala
// read another DataFrame
val df_sales: DataFrame = spark.read
  .option("header", "true")
  .option("delimiter", ",")
  .csv("src/main/resources/data/Walmart_sales.csv")

val df10 = df.select("availableText").union(df_sales.select("Weekly_Sales"))
```

**Be careful**: the only requirements for `.union()` is that the numbers of columns of the two DFs is the same, but it’s not required to be the same ones. This function resolvers columns **by name**, meaning that the first column of `df` is appended to the first column of `df_sales`, the second column of `df` is appended to the first column of `df_sales`, etc., regardless their name. Only the position matters. This means that, like in the example above, I’m connecting two columns (`availableText` and `Weekly_Sales`) from two different DFs that have nothing to do with each other. If we don’t pay attention we risk causing damage.

### `.unionByName()`

A safer approach is to use `unionByName()` function. This function resolver columns **by name**, meaning that the requirements is that the two DFs need to have the same columns, otherwise Spark return an error. But, if you want to union two DFs with different columns you can do:

```scala
val df10_2 = df.unionByName(df_sales, allowMissingColumns = true)
```

When using `allowMissingColumns = true` the result of the DataFrame contains `NULL` values for the columns that are missing on the DataFrame.

The result is:

![Untitled](images/Untitled%209.png)

As you can see, since there are no common columns between the two DFs, Spark add all the columns of `df_sales` that are missing on `df`.

## 2.10. Convert string to data and timestamp types `.to_date()` and `.to_timestamp()`

When we imported the csv file at the beginning of this post, we used `.option("inferSchema", "true")`. As I suggested in one of the previous post in this blog, it’s not suggested to use this option, but instead to use `.schema(csvSchema)` to manually assign a data type to each column. In our particular case, the `lastUpdated` column contains timestamp values, but if we print the schema we see that Spark assigns a string data type to that column. So functions `.to_date()` and `.to_timestamp()` allow us to convert a column in a data or timestamp data type. Let’s convert `lastUpdated` into both date and timestamp columns:

```scala
val df11 = df
  .withColumn("lastUpdated_timestamp", to_timestamp(col("lastUpdated"), "MMM dd, yyyy HH:mm:ss z"))
  .withColumn("lastUpdated_date", to_date(col("lastUpdated"), "MMM dd, yyyy HH:mm:ss z"))
```

Note that both functions require the pattern format of the string you want to convert.

Then let’s print the schema and the first 5 rows of `df11` (we’ll print only the columns we are considering):

![Untitled](images/Untitled%2010.png)

## 2.11. Convert date or timestamp to string `date_format()`

If you have a date or timestamp Column, you can convert it to a desirable string formats. Firstly, let’s create a new DF on which we’ll apply these functions:

```scala
val orders = Seq(
  Row("sandwich", "big", 10, "2024-03-24", "2024-03-24:14:31:20"),
  Row("pizza", "small", 15, "2024-03-22", "2024-03-22:21:00:12")
)
val schema = StructType(Seq(
  StructField("food", StringType, nullable = true),
  StructField("size", StringType, nullable = true),
  StructField("cost", IntegerType, nullable = true),
  StructField("order_date", StringType, nullable = true),
  StructField("order_timestamp", StringType, nullable = true)
))
val df_orders = spark.createDataFrame(spark.sparkContext.parallelize(orders), schema)

val df_orders_clean = df_orders
  .withColumn("order_date", to_date(col("order_date"), "yyyy-MM-dd"))
  .withColumn("order_timestamp", to_timestamp(col("order_timestamp"), "yyyy-MM-dd:HH:mm:ss"))

val df_date_formatted = df_orders_clean
    .withColumn("date_formatted", date_format(col("order_date"), "ddMMMyyyy"))
    .withColumn("timestamp_formatted", date_format(col("order_timestamp"), "ddMMMyyyy HH:mm:ss"))
```

Of course you can use whichever string format you want.

**Be careful**:

- the original column must be `date` or `timestamp` column!
- the resulting columns are `string` format, no longer `date` or `timestamp`!

Here’s the result:

![Untitled](images/Untitled%2011.png)

## 2.12. Extract a substring `.substring()`

`substring()` function is used to extract the substring from a DataFrame string column by providing the position and length of the string you wanted to extract. Let’s say we want to change `itemNumber` column by extracting only the first 5 characters:

```scala
val df12 = df
  .withColumn("itemNumber_v2", substring(col("itemNumber"), 0, 5))
```

Here’s the result:

![Untitled](images/Untitled%2012.png)

## 2.13. Remove white spaces `.trim()`

Let’s create a new DF just to highlight the problems we want to solve with `.trim()` function:

```scala
val orders_2 = Seq(
  (" sandwich  ", "big", 10, "2024-03-24"),
  ("   pizza", "small", 15, "2024-03-22"),
  ("salad  ", "small", 15, "2024-03-22")
)
val df13 = orders_2.toDF("food", "size", "cost", "order_date")
```

Note that food and size columns are not well formatted because values have some unwanted blank spaces at the beginning and at the end of values in some cases, and only at the beginning or only at the end in other cases. `.trim()` remove leading and trailing spaces:

```scala

val df13_2 = df13
  .withColumn("trim_food", trim(col("food")))
  .withColumn("ltrim_food", ltrim(col("food")))
  .withColumn("rtrim_food", rtrim(col("food")))
```

Note that in addition to `.trim()`, we also used `.ltrim()`, that removes only blank spaces at the beginning of the string, and `.rtrim()`, that removes blank spaces at the end of the string.

Here’s the result:

![Untitled](images/Untitled%2013.png)

## 2.14. Pad a string `.lpad()` and `.rpad()`

The `.lpad()` function used to left-pad a string column with a specified character or characters to reach a desired length. Considering that the maximum value of sold column is 1372 let’s say we want that all numbers must have 4 digits. This means that 1-digit numbers need three leading 0, 2-digit numbers need 2 leading zeros, 3-digit numbers need 1 leading zero:

```scala
val df17 = df.withColumn("lpad_sold", lpad(col("sold"), 4, "0"))
```

Here’s the result:

![Untitled](images/Untitled%2014.png)

The functioning of `.rpad()` is exactly the same, but the pad happens to the right-side.

## 2.15. Concatenate strings `.concat()` and `.concat_ws()`

Let’s say you want to create a new column by concatenating two or more existing columns. I have two options:

- using `.concat()`: concatenate two or more columns into a single new Column
- using `.concat_ws()`: concatenate two or more columns into a single new Column with a **separator**.

Let’s write a code that, theoretically, should return the same result for each function:

```scala
val tmp = Seq(
  ("sandwich", "big"),
  ("pizza", null),
  (null, "small"),
  (null, null)
)
val df18 = tmp.toDF("food", "size")
val df18_2 = df18
  .withColumn("concat", concat(col("food"), lit("_"), col("size")))
  .withColumn("concat_ws", concat_ws("_", col("food"), col("size")))
```

Here’s the result:

![Untitled](images/Untitled%2015.png)

**Be careful**:

- `.concat()` and `.concat_ws()` are equivalent only if the columns to be concatenated are not-null.
- if one of the two columns contains a null value:
    - `.concat()` produces a null value;
    - `.concat_ws()` produces a not-null value considering only not-null columns;
- if both columns contain null values:
    - `.concat()` produces a null value;
    - `.concat_ws()` produces a an empty string.

## 2.16. Replace part of string with another string `.regexp_replace()`

`.regexp_replace()` is a function that is used to replace part of a string (substring) value with another string on DataFrame column by using regular expression (regex). Let’s say I want to reaplace $ symbol with € symbol in `priceWithCurrency` column:

```scala
  val df19 = df
	  .withColumn("dollars_to_euros", regexp_replace(col("priceWithCurrency"), "US \\$", "EUR €"))

```

Note that we escape the `$` character using double backslashes **`\\`** because `$` is a special character in regular expressions.

Here’s the result:

![Untitled](images/Untitled%2016.png)

This function can be used to remedy the problem we encountered in the paragraph 2.12: we can replace an empty string with a null value:

```scala
val df18_final = df18_2
  .withColumn("concat_ws_final", when(col("concat_ws")==="", lit(null)).otherwise(col("concat_ws")))
```

Here’s the result:

![Untitled](images/Untitled%2017.png)

## 2.17. Create an array of strings `.split()`

The `.split()` method returns a new Column object that represents an array of strings. Each element in the array is a substring of the original column that was split using the specified pattern. Of course you can also create other columns extracting elements from that list or, for instance, concatenate them:

```scala
val df20 = df
  .withColumn("itemLocation_splitted", split(col("itemLocation"), ","))
  .withColumn("first_element", split(col("itemLocation"), ",")(0))
  .withColumn("second_element", split(col("itemLocation"), ",")(1))
  .withColumn("concat_elements", concat_ws("_", col("first_element"), col("second_element")))
```

![Untitled](images/Untitled%2018.png)

**Be careful**: since `,` is the delimiter, the `second_element` values starts with a blank space, so in the concat_elements column we’ll have `NY_ United States` instead of `NY_United States`. To fix it, we can simply use `.trim()` function:

```scala
val df20 = df
  .withColumn("itemLocation_splitted", split(col("itemLocation"), ","))
  .withColumn("first_element", split(col("itemLocation"), ",")(0))
  .withColumn("second_element", split(col("itemLocation"), ",")(1))
  .withColumn("second_element_fix", trim(split(col("itemLocation"), ",")(1)))
  .withColumn("concat_elements", concat_ws("_", col("first_element"), col("second_element")))
  .withColumn("concat_elements_fix", concat_ws("_", col("first_element"), col("second_element_fix")))
```

Here’s the result:

![Untitled](images/Untitled%2019.png)

---

# 3. Aggregation functions

Aggregation functions in Apache Spark are used to perform calculations across multiple rows of a DataFrame, typically grouped by one or more columns. These functions help summarize or aggregate data in various ways, such as calculating counts, sums, averages, minimum and maximum values, and more. 

Here are some common aggregation functions in Spark:

1. `count`: Counts the number of rows in a DataFrame.
2. `countDistinct`: Counts the number of distinct rows in a DataFrame.
3. `sum`: Calculates the sum of numerical values in a column.
4. `avg`: Calculates the average of numerical values in a column.
5. `min`: Finds the minimum value in a column.
6. `max`: Finds the maximum value in a column.
7. `agg`: Allows for aggregating multiple functions simultaneously.

**Be careful**: don’t forget to cast columns in the right way. Numerical aggregations work well only for numeric datatypes!

Grouping columns is done with `.groupBy()` function. Let’s say we want to count how many records we have for each `type` Column value:

```scala
val df21 = df
  .groupBy("type").count()
```

Here’s the result:

![Untitled](images/Untitled%2020.png)

Let’s complicate the exercise and let’s say we want now to know the `min`, `max`, and `avg` of `priceWithCurrency` column for each `type`. As we can note, `priceWithCurrency` column is a `string` Column in the form  like `US $2,054.83`. We cannot use numerical aggregations unless we convert this `string` into a `float` type.

```scala
val df21_2 = df
  .withColumn("price_fixed", regexp_replace(regexp_replace(col("priceWithCurrency"), "US \\$", ""), "\\,", "").cast("float"))
  .groupBy("type").agg(count("*"), min("price_fixed"), max("price_fixed"), avg("price_fixed"))
```

Here’s the result:

![Untitled](images/Untitled%2021.png)

Of course you can also group by more than one column:

```scala
val df21_3 = df
  .withColumn("price_fixed", regexp_replace(regexp_replace(col("priceWithCurrency"), "US \\$", ""), "\\,", "").cast("float"))
  .groupBy("seller", "type").agg(count("*"), min("price_fixed"), max("price_fixed"), avg("price_fixed"))
```

Here’s the result:

![Untitled](images/Untitled%2022.png)

---

# 4. Advanced functions

## 4.1. Window functions

Window functions in Spark are used to perform calculations across a set of rows related to the current row, often within a specified window of rows. These functions are applied to a group of rows defined by a **partition** and **ordered** by a specific column(s) **within that partition**. Let’s say we want to assign an ascending number starting from 1 to every partitions defined by the `seller` Column ordered by `lastUpdated` Column in descending order.

This means that:

- DataFrame will be split into partitions and each partition will contain all rows associated to each value of `seller`;
- rows in each partition are ordered in descending order according to `lastUpdated` Column;
- at each record a number starting from 1 is assigned.

This operation will be obtained with `.row_number()` functions associated to a `Window` object carefully partitioned and ordered as decided above.

Let’s make this example (remember we have first to cast `lastUpdated` to a `timestamp` column since now it’s a `string` column) by also including the functions `.rank()` and `.dense_rank()` for example purposes only:

```scala
val windowSpec = Window.partitionBy("seller").orderBy(desc("lastUpdated_timestamp"))

val df22 = df
  .withColumn("lastUpdated_timestamp", to_timestamp(col("lastUpdated"), "MMM dd, yyyy HH:mm:ss z"))
  .withColumn("row_number", row_number().over(windowSpec))
  .withColumn("rank", rank().over(windowSpec))
  .withColumn("dense_rank", dense_rank().over(windowSpec))
  .orderBy(col("seller"), desc("lastUpdated_timestamp"))
```

Here’s the result:

![Untitled](images/Untitled%2023.png)

## 4.2. `.distinct()` vs `.dropDuplicates()`

These two functions are not really advanced functions, but I put them in this category because their use could be a little bit tricky (actually only `.dropDuplicates()` is tricky).

Both functions are used to remove duplicate rows from a DataFrame. The difference is that:

- `.distinct()` operates on **all columns** in the DF;
- `.dropDuplicates()` can operate both on **entire DF** or on a **subset of columns**, by specifying them as arguments in string format.

Let’s make these examples and we’ll analyze one by one:

```scala
val df12 = df.distinct()
val df12_2 = df.select("type").distinct()

val df13 = df.dropDuplicates()
val df13_2 = df.dropDuplicates("itemLocation", "seller")
```

- `df12` and `df13` will be exactly the same.
- `df12_2` returns only the distinct elements of the column `type`:
    
    ![Untitled](images/Untitled%2024.png)
    
- `df13_2` is the tricky one: `.dropDuplicates()` will only consider the `itemLocation` and `seller` columns when identifying duplicates, and it will keep a **random occurrence of each duplicate**. This is the tricky part. When the DF is split across partitions, you don’t know which occurrence Spark decides to keep (maybe there is an order, but honestly I didn’t find it). `.dropDuplicates()` keeps the first occurrence only if there is 1 partition. So be careful when using this function.

We can remedy this problem by using a window function, specifically a `.row_number()` function. What we have to do is force the sorting we want so that we take the specific record we are interested in and eliminate all other duplicates:

```scala
val windowSpec2 = Window.partitionBy("itemLocation", "lastUpdated_timestamp").orderBy("lastUpdated_timestamp")
val df_24_2_fix = df_clean
  .withColumn("row_number", row_number().over(windowSpec2))
  .filter(col("row_number")===1)
```

- we created partitions by grouping by `itemLocation` and `lastUpdated_timestamp`, and ordering by `lastUpdated_timestamp`. In this way we forced the sorting so that we are sure not to remove duplicates randomly;
- to each record in each partition we assigned an ascending integer, in particular we assign `1` to each record we want to keep in the new DF;
- only records with a value of `1` are taken in each partition.