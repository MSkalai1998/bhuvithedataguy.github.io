---
layout: post
date: 2020-11-07 11:30:00 +0530
title: Airflow BigQueryHook And Operator To Create Range Partition
description: Airflow BigQuery Operator packed with Time partition parameter. But it doesn't have the parameter for the Range Partition. But alternatively, we can use `table_resource` parameter to create the Range partition.
categories:
- Airflow
tags:
- gcp
- airflow
- bigquery
- python
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/airflow/airflow.png"
---
**Airflow BigQuery Operator packed with Time partition parameter. But it doesn't have the parameter for the Range Partition. But alternatively, we can use `table_resource` parameter to create the Range partition.** 

If you are already familiar with Airflow bigquery operator or bigquery hook, then you might notice that it won't support create tables with Range Partition, but it has a separate parameter from time partition. And I saw some folks were already coming up with this situation and they solved it by customizing the BigQuery Operator to support Range Partition support. There is a Pull Request has already given in Airflow's Github repo. But unfortunately, it didn't merge with the airflow. Because it's better to minimize the operator interface. And the airflow maintainers suggested using the `table_resource` parameter to overcome this problem. Also, they mentioned, even the time partition parameter will be removed in future releases. 

I tried to use the table resource option to create the table with Range partition. Unfortunately, it didn't work as I expected.  It just created the table without any columns and no partitions. Then I started reading more about the table resource's behavior in BigQuery API. 

The table resource is a JSON representation of a BigQuery table, It has all the properties to define a proper table. Then I realized why it didn't create any columns when I tried this very first time. Also, I got help from the Airflow maintainers to understand better this table_resource parameter in BigQueryOperator. 

Let's see what the operator is saying about this parameter.
```py
:param table_resource: Table resource as described in documentation:
https://cloud.google.com/bigquery/docs/reference/rest/v2/tables#Table
        If provided all other parameters are ignored.
```
The description gives the meaning of the parameter. Generally, we pass the table in a JSON file or a parameter into the `BigQueryHook.create_empty_table` operator. But if we are going to use the `table_resource` then all other table related parameters will be ignored like schema, time partition, etc.  So all the properties for the table have to be passed into the table_resource parameter.  To read more about all the properties that are supported by the table resource, [hit this link](https://cloud.google.com/bigquery/docs/reference/rest/v2/tables).

## Code:

This is my sample code with BigQueryHook to create an empty BigQuery table with Range partition.
<script src="https://gist.github.com/BhuviTheDataGuy/aaa85b2cf55e1be416b8ab4c26a9335b.js"></script>