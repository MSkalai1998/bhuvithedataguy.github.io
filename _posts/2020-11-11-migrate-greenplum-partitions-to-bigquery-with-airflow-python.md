---
layout: post
date: 2020-11-11 07:30:00 +0530
title: Migrate Greenplum Partitions To BigQuery With Airflow/Python
description: Migrate the Greenplum partition table to BigQuery. It creates an empty table on BigQuery along with partitions like Greenplum.
categories:
- Airflow
tags:
- gcp
- airflow
- bigquery
- python
- greenplum
- postgresql
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/airflow/Migrate Greenplum Partitions To BigQuery With Airflow Python2.png"
---
**Migrate the Greenplum partition table to BigQuery. It creates an empty table on BigQuery along with partitions like Greenplum.**

We have many airflow operators to help migrate or sync the data to Google BigQuery. If the target table is not there, then some operators like GCS to BigQuery operators will automatically create the table using the schema object. But these operators will just migrate the table DDL without the partition information. During any DataWarehouse migration, we have to keep the table's partitions as well. In this blog, Im sharing my experience with migrating some of the Greenplum tables DDL to BigQuery with partitions.

## Greenplum Partitions:

Greenplum supports two different partitions.

* range partitioning: division of data based on a numerical range, such as date or price.
* list partitioning: division of data based on a list of values, such as sales territory or product line.
* Also, we use a combination of both types.

You can use [pg_partitions](http://docs.greenplum.org/5140/ref_guide/system_catalogs/pg_partitions.html) system table to see the detailed view of the table partitions.

## BigQuery Partitions:

BigQuery supports 3 different types of partitions.

* Ingestion time: Tables are partitioned based on the data's ingestion (load) time or arrival time.
* Date/timestamp/datetime: Tables are partitioned based on a TIMESTAMP, DATE, or DATETIME column.
* Integer range: Tables are partitioned based on an integer column.

If you compare the BigQuery partition capability with Greenplum, all the BigQuery partitions are equivalent to the Range partition in Greenplum. 

1. So it won't support the list partition.
2. A combination of List and Range is not possible.
3. Only one level of partitions are supported (or subpartitions are not possible)

## Find the partition details:

Considering the above 3 points, we'll convert the Greenplum partitions with the following steps. 

1. Scan the giving table name in the pg_partitions table and find whether it is partitioned or not. Since we are interested in Range partition, so we'll check whether this table has Range partition or not.
2. Then the next step is to find the column that is used for partitioning. 
3. In Greenplum, it is just a range partition but in BigQuery we have to mention whether it is an integer range or DateTime. So we have to find the data type of the partitioned column, if it is `int` then its Integer partition, or if it is `timestamp or timestampz` then its date time partition.
4. For the timestamp based columns, we have to mention the time unit like partition based on HOUR/DAY/MONTH/YEAR. So again in the `pg_partitions` we'll get the information in `partitioneveryclause` column.
5. If it is an integer column then find the start and end values using `partitionrangestart` and `partitionrangeend` columns and the interval in `partitioneveryclause` column.
 
I created a python function to combine all these logics and generate information about the source partition.

## Extract Table DDL:

Now we have to extract each column and its data type for the given table, then we have to map the equivalent  BigQuery data type. All the information has to be returned as JSON and it will be used as the [BigQuery schema object](https://cloud.google.com/bigquery/docs/schemas). I used [information_schema.columns](https://www.postgresql.org/docs/13/infoschema-columns.html) to identify the column names and its data type. Here is what I have used for the data type mapping.

{% include lazyload.html image_src="/assets/airflow/Migrate Greenplum Partitions To BigQuery With Airflow Python1.png" image_alt="Migrate Greenplum Partitions To BigQuery With Airflow Python" image_title="Migrate Greenplum Partitions To BigQuery With Airflow Python" %}

## Generate the BigQuery Partition details:

In airflow Bigquery hook, [it won't support the Range partition as a parameter, but we can use table_resource to pass the whole table's properties](https://thedataguy.in/airflow-bigqueryhook-operator-to-create-range-partition/).  So the next step is to consolidate all the partition information and generate the JSON format. Here we are combining the schema and the partition details, then it'll be passed as the `table_resource` in the BigQuery API.

{% include lazyload.html image_src="/assets/airflow/Migrate Greenplum Partitions To BigQuery With Airflow Python2.png" image_alt="Migrate Greenplum Partitions To BigQuery With Airflow Python" image_title="Migrate Greenplum Partitions To BigQuery With Airflow Python" %}

## Create the table via BigQueryHook:

And we are going to use the BigQuery Hook (from the backport providers) to create the empty table using the table resource. If you are using an older version of Airflow(libraries from `airflow.contrib` then the hook doesn't support the table_resource option). 

You can use your own python code instead of airflow by just replacing this step. And here is my final version of the complete code.

<script src="https://gist.github.com/BhuviTheDataGuy/efd142be0cbe8c3935c50db34dfe9df7.js"></script>