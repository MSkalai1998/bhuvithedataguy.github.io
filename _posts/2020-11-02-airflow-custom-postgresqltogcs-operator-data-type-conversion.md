---
layout: post
date: 2020-11-02 08:20:00 +0530
title: Airflow Custom PostgreSQL To Google Cloud Storage Operator
description: A custom PostgresToGCSOperator that has proper data type mapping and solve the conversion DATE, TIME and DateTime datatype to epoch timestamp conversion. 
categories:
- Airflow
tags:
- gcp
- airflow
- postgresql
- greenplum
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/python/Airflow Custom PostgreSQL To Google Cloud Storage Operator2.png"
---
The default PostgresToGoogleCloudStorageOperator[(v1.*)](https://airflow.apache.org/docs/stable/_modules/airflow/contrib/operators/postgres_to_gcs_operator.html) and PostgresToGCSOperator[(v2.0)](https://github.com/apache/airflow/blob/438547d6e6ab3db3c94b9385a01ef4fcd63af9d7/airflow/providers/google/cloud/transfers/postgres_to_gcs.py) are converting any date, time, timestamp, DateTime values to `epoch timestamp` format. Also, the schema file will be mapped to TIMESTAMP and very few data types are supported. Recently I was working on a data pipeline where I need to sync the data from Greenplum data warehouse to GCP BigQuery. So the tables will be automatically created based on the schema file that is exported by the PostgresToGCSOperator. But unfortunately, all of our TIME and Date columns are converted to Epoch format and this needs a lot of changes in the reporting service. 

Also, we noticed that the tables that are created automatically by the schema file are having only 4 data types. This made us think instead of fixing the conversion type on the Reporting service, we have to take a look at the operator and fix them on the airflow side itself. 

Let's take a look at the PostgresToGCSOperator data type mapping.

{% include lazyload.html image_src="/assets/python/Airflow Custom PostgreSQL To Google Cloud Storage Operator0.png" image_alt="Airflow Custom PostgreSQL To Google Cloud Storage Operator" image_title="Airflow Custom PostgreSQL To Google Cloud Storage Operator" %}

The list of keys inside the `type_map` is referring to the PostgreSQL data type. But instead of using data type names, it has the OID of the data types. PostgreSQL supports so many data types but this operator is having a very limited set of data types. 

On the other hand, only 4 BigQuery data types are mentioned. So except these data types, everything will be converted to `STRING` data type. So we decided to use what are all the data types that we need for our use case. Then we come up with the list of data type mapping as mentioned below. Rest of the data types will be considered as `STRING`. 

{% include lazyload.html image_src="/assets/python/Airflow Custom PostgreSQL To Google Cloud Storage Operator1.png" image_alt="Airflow Custom PostgreSQL To Google Cloud Storage Operator" image_title="Airflow Custom PostgreSQL To Google Cloud Storage Operator" %}

But still, this is data type mapping only, it won't completely solve the problem. It worked for `CSV` export, but for JSON export, it was not able to find We have to fix the conversion part as well to get the timestamp and Data, Time values as it is. In the operator, we found the piece of line that is doing this conversion.

Then we replaced the conversion function with the following code.

{% include lazyload.html image_src="/assets/python/Airflow Custom PostgreSQL To Google Cloud Storage Operator2.png" image_alt="Airflow Custom PostgreSQL To Google Cloud Storage Operator" image_title="Airflow Custom PostgreSQL To Google Cloud Storage Operator" %}

Finally, this is our customized PostgresToGCSOperator operator.

<script src="https://gist.github.com/BhuviTheDataGuy/532f87214b99ad66b65c377b07fc7c57.js"></script>

## Conclusion:

Please remember, we are converting these data types as per our need. And almost all the data types are converted here except the special data types. If you want, you can also customize this operator with more data types. 