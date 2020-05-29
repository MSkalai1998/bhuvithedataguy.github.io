---
title: RedShift Dynamic WLM With Lambda
date: 2020-05-29 19:45:00 +0530
description: Dynamically change the RedShift WLM configurations with AWS Lambda. You can dynamically change both manual and auto WLM without any downtime. 
categories:
- RedShift
tags:
- aws
- redshift
- lambda
- python
- wlm
image: "/assets/RedShift Dynamic WLM With Lambda.jpg"
---
Redshift doesn't support Dynamic WLM natively. we have both Manual and Auto WLM. Auto WLM will be allocating the resources and the concurrency dynamically based on past history. Its using ML algorithms internally to allocate the resources. It's a very good choice for a standard cluster like not much difference in the workload. They follow the same pattern as night time ETL, morning BI users, and so on. But if you want to dynamically change the Memory and the Concurrency for a manual WLM then you use AWS Lambda. There is a solution already available on AWS's RedShift utilities, but its not a sperate package. It comes with many other things. If you want to setup your own dynamic WLM, then this blog will help you. 

## How AWS handles this dynamic WLM? 

> From AWS docs,
In each queue, WLM creates a number of query slots equal to the queue's concurrency level. The amount of memory allocated to a query slot equals the percentage of memory allocated to the queue divided by the slot count. If you change the memory allocation or concurrency, Amazon Redshift dynamically manages the transition to the new WLM configuration. Thus, active queries can run to completion using the currently allocated amount of memory. At the same time, Amazon Redshift ensures that total memory usage never exceeds 100 percent of available memory.

So we'll never face any downtime while changing this. 

## Timing for the allocation:

We are using manual WLM, and we know the workload very well. I had a requirement that all of the ETL processes are running from 12 AM to around 6 AM. So I want to allocate almost all the memory to the ETL users group. Then After 8 AM to 6 PM, it is heavily used by BI users. 

* ** 6 PM - 8 AM ** 90% memory to ETL users.
* ** 8 AM - 6 PM **  90% memory to BI users.

So I need to trigger the lambda function 2 times in a day. I don't want to use 2 different lambda functions for this. So Im my lambda function, I'll get the current hour,  based on that it'll decide when configuration should be applied. 

> You can use the same logic for Auto WLM as well to change the priority. 

## Create config files:

I recommend you that instead of manually typing this configuration values, just create a new parameter group with your queues, QMR rules and etc. Then you can get the JSON content from the WLM window. Just copy that and upload it to the S3 bucket. Similarly, one config file the next set of config and upload to S3. Here are my config files. 

** For ETL users: ** power-etl.json
{% highlight json%}
[{
	"query_concurrency": 10,
	"memory_percent_to_use": 90,
	"query_group": [],
	"query_group_wild_card": 0,
	"user_group": ["etlusers"],
	"user_group_wild_card": 0
}, {
	"query_concurrency": 2,
	"memory_percent_to_use": 5,
	"query_group": [],
	"query_group_wild_card": 0,
	"user_group": ["biusers"],
	"user_group_wild_card": 0
}, {
	"query_concurrency": 2,
	"memory_percent_to_use": 5,
	"query_group": [],
	"query_group_wild_card": 0,
	"user_group": [],
	"user_group_wild_card": 0
}, {
	"query_group": [],
	"query_group_wild_card": 0,
	"user_group": [],
	"user_group_wild_card": 0,
	"auto_wlm": false
}, {
	"short_query_queue": true
}]
{% endhighlight %}

** For BI users: ** power-bi.json
{% highlight json%}
[{
	"query_concurrency": 2,
	"memory_percent_to_use": 5,
	"query_group": [],
	"query_group_wild_card": 0,
	"user_group": ["etlusers"],
	"user_group_wild_card": 0
}, {
	"query_concurrency": 10,
	"memory_percent_to_use": 90,
	"query_group": [],
	"query_group_wild_card": 0,
	"user_group": ["biusers"],
	"user_group_wild_card": 0
}, {
	"query_concurrency": 2,
	"memory_percent_to_use": 5,
	"query_group": [],
	"query_group_wild_card": 0,
	"user_group": [],
	"user_group_wild_card": 0
}, {
	"query_group": [],
	"query_group_wild_card": 0,
	"user_group": [],
	"user_group_wild_card": 0,
	"auto_wlm": false
}, {
	"short_query_queue": true
}]
{% endhighlight %}

## S3 path:

{% highlight sh%}
s3://bucketname/dynamic-wlm/power-etl.json
s3://bucketname/dynamic-wlm/power-bi.json
{% endhighlight %}

## Lambda function:

This is very simple, and your just need the following IAM role to this Lambda function. No VPC access and set the timeout to 1min.
1. AmazonS3ReadOnlyAccess
2. AWSLambdaBasicExecutionRole
3. An inline policy with ModifyClusterParameterGroup (refer my policy below)

{% highlight json%}
-- Replace 
-- 00000000000000 with your account ID
-- us-east-1 - Your RedShift region
-- manual-wlm - Parameter group name

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "redshift:ModifyClusterParameterGroup",
            "Resource": "arn:aws:redshift:us-east-1:00000000000000:parametergroup:manual-wlm"
        }
    ]
}
{% endhighlight %}

## Lambda Code: 

Please replace region and bucket. 

{% highlight python%}
import json
import boto3
import datetime

region='us-east-1'
bucket='bucketname'
current_timestamp = datetime.datetime.now()
hour=current_timestamp.strftime("%H")


if hour == '06':
    config = 'power-etl'
elif hour == '18':
    config = 'power-bi'
else:
    print("Wrong timing")
    exit()

redshift_client = boto3.client('redshift', region_name=region)
s3_client = boto3.client('s3', region_name=region)
obj = s3_client.get_object(Bucket=bucket, Key='dynamic-wlm/'+config+'.json')
config_body = obj['Body'].read()
wlm_config = json.dumps(json.loads(config_body))

def lambda_handler(event, context):
	redshift_client.modify_cluster_parameter_group(
	        ParameterGroupName='manual-wlm',
	        Parameters=[
	            {
	                'ParameterName': 'wlm_json_configuration',
	                'ParameterValue': wlm_config,
	                'Description': 'ruleset_name',
	                'Source': 'user',
	                'ApplyType': 'dynamic',
	                'IsModifiable': True
	            },
	        ])
{% endhighlight %}

If you don't want to use S3, instead if you want to try with line, then remove the followling 3 lines.

{% highlight python%}
s3_client = boto3.client('s3', region_name=region)
obj = s3_client.get_object(Bucket=bucket, Key='dynamic-wlm/'+config+'.json')
config_body = obj['Body'].read()
{% endhighlight %}

Convert your JSON content into a single line.

{% highlight python%}
config_body = """ JSON FILE CONTENT IN ONE LINE """
{% endhighlight %}

Then change the time based logic as per your need. Now can add a cloudwatch trigger to trigger this twice in a day.


