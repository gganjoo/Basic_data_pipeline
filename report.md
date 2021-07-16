# Project 2 report
# Gauri Ganjoo

## Project Repo
yml file, json file, report.md, spark history, command line history

## Setup

Opens containers for zookeeper, mids, spark, kafka, and cloudera. (yml file originally provided by w205)
```
docker-compose up -d
```

Creates a topic called assessment in kafka to temporarily store data
```
docker-compose exec kafka kafka-topics --create --topic assessment --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
```

Imports raw data into the kafka topic assessment
```
 docker-compose exec mids bash -c "cat /w205/project-2-gganjoo/assessment-attempts-20180128-121051-nested.json | jq '.[]' -c | kafkacat -P -b kafka:29092 -t assessment"
```

Starts pyspark, which will be used to format some of the data.
```
docker-compose exec spark pyspark
```
## Spark

Imports needed packages
```
import json
from pyspark.sql import Row
```

Reads in data from kafka to spark
```
raw_data = spark.read.format("kafka").option("kafka.bootstrap.servers", "kafka:29092").option("subscribe","assessment").option("startingOffsets", "earliest").option("endingOffsets", "latest").load() 
```

Caches data to help computer recall and prevent errors down the line
```
raw_data.cache()
```

Transforms values into strings to help with querying
```
data = raw_data.selectExpr((("CAST(value as STRING)")))
```

Encoding into utf8(to help recognize all/most characters)
```
import sys
sys.stdout = open(sys.stdout.fileno(), mode ='w', encoding ='utf8', buffering =1)

```

Converts raw data from json to dataframe format.
```
extracted_data = data.rdd.map(lambda x: Row(**json.loads(x.value))).toDF()
```

Registers data as temporary data so that sql queries can be done on it.
```
extracted_data.registerTempTable('assessments')
```

## SQL Table structure
All of the code in this section was run in spark and should work if all the above lines of code were run first.

```
spark.sql('select * from assessments limit 1').show()
```
```
+--------------------+-------------+--------------------+-----------------+--------------------+-----------------+------------+--------------------+--------------------+--------------------+
|        base_exam_id|certification|           exam_name|  keen_created_at|             keen_id|   keen_timestamp|max_attempts|           sequences|          started_at|        user_exam_id|
+--------------------+-------------+--------------------+-----------------+--------------------+-----------------+------------+--------------------+--------------------+--------------------+
|37f0a30a-7464-11e...|        false|Normal Forms and ...|1516717442.735266|5a6745820eb8ab000...|1516717442.735266|         1.0|Map(questions -> ...|2018-01-23T14:23:...|6d4089e4-bde5-4a2...|
+--------------------+-------------+--------------------+-----------------+--------------------+-----------------+------------+--------------------+--------------------+--------------------+
```
There are ten columns in the data table. Note that the sequences column is filled with a map function, so there is more information there than initially appears. As it is,sql queries involving the equences column do not tell us much because the data in it is still nested.

I would use the base_exam_id to identify unique assessments when querying over the exam_name column because there are more unique base_exam_ids than names. 

```
spark.sql('select count(distinct base_exam_id), count(distinct exam_name) from assessments').show()
```
```
+----------------------------+-------------------------+
|count(DISTINCT base_exam_id)|count(DISTINCT exam_name)|
+----------------------------+-------------------------+
|                         107|                      103|
+----------------------------+-------------------------+
```
The data team may want to fix the descrepency at a later date. Also note that below when I identified the most popular and least popular courses, I used the exam_name as the identifier to make the report more human readable and easier to understand(since id numbers don't mean much to the human eye). For a different report I would have used the base_exam_id. 

```
spark.sql('select distinct max_attempts from assessments').show()
```
```
+------------+
|max_attempts|
+------------+
|         1.0|
+------------+
```
I also wanted to note that right now all entries in the max_attempts column are 1.0. I do not know if the team was expecting this or plans to do something with it later, but I wanted to point this out in case it was not planned. 

```
assessments.printSchema()
```
```
root
 |-- base_exam_id: string (nullable = true)
 |-- certification: string (nullable = true)
 |-- exam_name: string (nullable = true)
 |-- keen_created_at: string (nullable = true)
 |-- keen_id: string (nullable = true)
 |-- keen_timestamp: string (nullable = true)
 |-- max_attempts: string (nullable = true)
 |-- sequences: map (nullable = true)
 |    |-- key: string
 |    |-- value: array (valueContainsNull = true)
 |    |    |-- element: map (containsNull = true)
 |    |    |    |-- key: string
 |    |    |    |-- value: boolean (valueContainsNull = true)
 |-- started_at: string (nullable = true)
 |-- user_exam_id: string (nullable = true)
```

Above is the schema/datatypes for the table. You can get a better idea of what the structure of the sequences column looks like. I also want to note that I did not change the time related variables since I was unsure of what format the data tem would ultimately use. 

## Questions

1. How many assessments are in the dataset? 

```
spark.sql("select count(distinct base_exam_id) from assessments").show()
```
```
+----------------------------+                                                  
|count(DISTINCT base_exam_id)|
+----------------------------+
|                         107|
+----------------------------+
```

2. How many people took *Learning Git*? 
```
spark.sql("select count(base_exam_id) from assessments where base_exam_id in (select distinct base_exam_id from assessments where exam_name IN ('Learning Git'))").show()
```
```
+-------------------+
|count(base_exam_id)|
+-------------------+
|                394|
+-------------------+
```

4. What is the least common course taken? And the most common?

```
spark.sql('select count(exam_name) as popularity ,exam_name from assessments group by exam_name order by popularity desc limit 1').show()
spark.sql('select count(exam_name) as popularity ,exam_name from assessments group by exam_name order by popularity asc limit 1').show()
```
```
+----------+------------+
|popularity|   exam_name|
+----------+------------+
|       394|Learning Git|
+----------+------------+
+----------+--------------------+
|popularity|           exam_name|
+----------+--------------------+
|         1|Learning to Visua...|
+----------+--------------------+
```

## Saving data and closing machines


Saves the datatable to hadoop for so that others can query table in future.
```
extracted_data.write.parquet("/tmp/assessments")
```

Exits out of pyspark back to terminal. Takes down docker containers.
```
exit()
docker-compose down
```



## What did not work

The sequences column is not queriable via sql as is. It is a column with a map structure that I was unable to flatten in python. I have attached a spark-history txt file in this repo with some of the coding I tried to do to flatten the map format.  