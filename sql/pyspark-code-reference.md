# PySpark References

Check here for SQL reference: https://spark.apache.org/docs/latest/api/sql/index.html
- Context

```python
import pyspark
from pyspark import SparkContext
from pyspark.sql import SparkSession
from pyspark.sql import SQLContext

# create a SparkSession instance with the name moviedb with Hive support enabled
# https://spark.apache.org/docs/latest/sql-data-sources-hive-tables.html
spark = SparkSession.builder.appName("moviedb").enableHiveSupport().getOrCreate()

# create a SparkContext instance which allows the Spark Application to access 
# Spark Cluster with the help of a resource manager which is usually YARN or Mesos
sc = SparkContext.getOrCreate()

# create a SQLContext instance to access the SQL query engine built on top of Spark
sqlContext = SQLContext(spark)
```
- Dataset

https://www.kaggle.com/rounakbanik/the-movies-dataset

```
# set the file_path variable in the beginning of the file
# or if your Spark application interacts with other applications, parameterize it
file_path = 'datasets/moviedb/movies_metadata.csv'

# method 1 for reading a CSV file
df = spark.read.csv(file_path, header=True)

# method 2 for reading a CSV file
df = spark.read.format(csv_plugin).options(header='true', inferSchema='true').load(file_path)
```

- Reading ways

```python
# Reading a csv file - all of these methods work the same for all the different formats
df = spark.read.csv(csv_file_path)
df = spark.read.format('csv').options(header=True,inferSchema=True).load(csv_file_path)
df = spark.read.format('csv').options(header='True',inferSchema='True').load(csv_file_path)
df = spark.read.format('CSV').options(header='true',inferSchema='true').load(csv_file_path)
df = spark.read.csv(file_path, header=True)
df = spark.read.csv(file_path, header='true')

# Reading a json file
df = spark.read.json(json_file_path)

# Reading a text file
df = spark.read.text(text_file_path)

# Reading a parquet file
df = spark.read.load(parquet_file_path) # or
df = spark.read.parquet(parquet_file_path)

# Reading a delta lake file
df = spark.read.format("delta").load(delta_lake_file_path)
```

- Writing Data

```python
# Write file to disk in parquet format partitioned by year - overwrite any existing file
df.write.partitionBy('year').format('parquet').mode('overwrite').save(parquet_file_path)

# Write file to disk in parquet format partitioned by year - append to an existing file
df.write.partitionBy('year').format('parquet').mode('append').save(parquet_file_path)

# Write data frame as a Hive table
df.write.bucketBy(10, "year").sortBy("avg_ratings").saveAsTable("films_bucketed")
```

- Creating DataFrames

```python
from pyspark.sql import Row

# populate two rows with random values
f1 = Row(original_title='Eroica', budget='13393950', year=1992)
f2 = Row(original_title='Night World', budget='1255930', year=1998)

# store the two rows in an array and pass it to Spark
films = [f1, f2]
df = spark.createDataFrame(films)

df.show()
```

```python
rdd = spark.textFile(csv_file_path)

from pyspark.sql.types import StringType, StructField, StructType, IntegerType
schema = StructType([
        StructField("first_name", StringType(), True),
        StructField("last_name", StringType(), True),
        StructField("age", IntegerType(), True)
    ])
   
df = spark.createDataFrame(rdd, schema)
```

- Modifying DataFrames

```python
# Create a column with the default value = 'xyz'
df = df.withColumn('new_column', F.lit('xyz'))

# Create a column with default value as null
df = df.withColumn('new_column', F.lit(None).cast(StringType()))

# Create a column using an existing column
df = df.withColumn('new_column', 1.4 * F.col('existing_column'))

# Another example using the MovieLens database
df = df.withColumn('test_col3', F.when(F.col('avg_ratings') < 7, 'OK')\
                                 .when(F.col('avg_ratings') < 8, 'Good')\
                                 .otherwise('Great')).show()

# Create a column using a UDF

def categorize(val):
  if val < 150: 
    return 'bucket_1'
  else:
    return 'bucket_2'
    
my_udf = F.udf(categorize, StringType())

df = df.withColumn('new_column', categorize('existing_column'))
```

```
# Changing column name with withColumnRenamed feature
df = df.withColumnRenamed('existing_column_name', 'new_column_name')

# Changing column with selectExpr (you'll have to select all the columns here)
df = df.selectExpr("existing_column_name AS existing_1", "new_column_name AS new_1")

# Changing column with sparksql functions - col and alias
from pyspark.sql.functions import col
df = df.select(col("existing_column_name").alias("existing_1"), col("new_column_name").alias("new_1"))

# Changing column with a SQL select statement
sqlContext.registerDataFrameAsTable(df, "df_table")
df = sqlContext.sql("SELECT existing_column_name AS existing_1, new_column_name AS new_1 FROM df_table")
```

```python
# Remove a column from a DataFrame
df.drop('this_column')

# Remove multiple columns in a go
drop_columns = ['this_column', 'that_column']
df.select([col for col in df.columns if column not in drop_columns])
```

- Joins
```
# Joining two DataFrames
df1.join(df2, 'title', 'full')

# Another way to join DataFrames
df1.join(df2, 'title', how='left') 

# Cross join when you don't specify a key
df1.join(df2)

# Another way to join
df1.join(df2, df1.title == df2.title, how='left')

# PySpark supports lesser known join types such as semi left and anti left
df1.join(df2, on=['title'], how='left_anti')
df1.join(df2, on=['title'], how='left_semi')
```

- Filters

```python
# Filter movies with avg_ratings > 7.5 and < 8.2
df.filter((F.col('avg_ratings') > 7.5) & (F.col('avg_ratings') < 8.2)).show()

# Another way to do this
df.filter(df.avg_ratings.between(7.5,8.2)).show()
```

```python
# Finding info of Ace Ventura films
df.where(F.lower(F.col('title')).like("%ace%")).show()

# Another way to do this
df.where("title like '%ace%'").show()

# Using where clause in sequence
df.where(df.year != '1998').where(df.avg_ratings >= 6.0)
```

```python
# Find all the films for which budget information is not available
df.where(df.budget.isNull()).show()

# Similarly, find all the films for which budget information is available
df.where(df.budget.isNotNull()).show()
```

- Aggregates

```
# Year wise summary of a selected portion of the dataset
df.groupBy('year')\
          .agg(F.min('budget').alias('min_budget'),\
               F.max('budget').alias('max_budget'),\
               F.sum('revenue').alias('total_revenue'),\
               F.avg('revenue').alias('avg_revenue'),\
               F.mean('revenue').alias('mean_revenue'),\
              )\
          .sort(F.col('year').desc())\
          .show()

# Pivot to convert Year as Column name and Revenue as the value
df.groupBy().pivot('year').agg(F.max('revenue')).show()
```

- Sorting
```
df.filter(df.year != '1998').sort(F.asc('year'))
df.filter(df.year != '1998').sort(F.desc('year'))
df.filter(df.year != '1998').sort(F.col('year').desc())
df.filter(df.year != '1998').sort(F.col('year').asc())

df.filter(df.year != '1998').orderBy(F.asc('year'))
df.filter(df.year != '1998').orderBy(F.desc('year'))
df.filter(df.year != '1998').orderBy(F.col('year').desc())
df.filter(df.year != '1998').orderBy(F.col('year').asc())
```

- Window

```python
from pyspark.sql import Window

# Rank all the films by revenue in the default ascending order
df.select("title", "year", F.rank().over(Window.orderBy("revenue")).alias("revenue_rank")).show()

# Rank year-wise films by revenue in the descending order
df.select("title", "year", F.rank().over(Window.partitionBy("year").orderBy("revenue").desc()).alias("revenue_rank")).show()
```
