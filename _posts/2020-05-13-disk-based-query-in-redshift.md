---
layout: post
date: 2020-05-13 11:20:00 +0530
title: Disk Based Queries -  A Hidden Monster in RedShift
description: How to find disk based queries in redshift and solve them. Also how get how much space used by a query in RedShift with stl_query and svl_query_summary tables.
categories:
- RedShift
tags:
- aws
- redshift
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/Disk Based Queries in RedShift.jpg"
---
Disk based queries in RedShift are a common thing. Whenever the allocated memory is not enough then RedShift will do disk based operations. Generally its data warehouse, we may be allocated multi Terabytes of space to the disk. So a lot of RedShift customers will never consider that the disk bases queries as a bottleneck. But when you have a huge size of data in the cluster and the running a very complex SQL query with a lot of joins and aggregations then you may consider this a problem. But at the same time, you have free space up to a few TB, then this issue will turn into a monster. 

## Why disk based queries? 

RedShift is operating with the WorkLoad management system to allocate the number of concurrent slots and allocate the memory for each slot. If you are performing a Join or Aggregate operation then the RedShift will pick the columns from the disk to in-memory and do the whole process. But if the memory which is allocated for this slot is not enough or the Queue doesn't have enough free slots then RedShift will move this particular operation to Disk and complete it. The downside of this process is, performing any disk bases operation comparing with memory is very slow. Also, it may eat your available disk IOPS. 

## The Monster:

Recently I was working with a large dataset on RedShift where I have 1.5TB free space. A new complex SQL query starts to execute and it's killed within 10mins. I didn't know this then I got an alert notification that my disk was full. When I check the disk used graph from the console, the utilization went to 100%. Im sure that my ETL and other write operations are already done. Only read workloads are going on. So I wanted to take a look at all the queries that were running at this particular time. I filtered the queries bases on disk based from the `svl_query_summary`. This reveals who is the real monster in my cluster.

![Disk Based Queries in RedShift](/assets/Disk Based Queries in RedShift2.jpg "Disk Based Queries in RedShift"){:class="lazyload"}

![Disk Based Queries in RedShift](/assets/Disk Based Queries in RedShift1.jpg "Disk Based Queries in RedShift"){:class="lazyload"}

See more than 3TB disk space used to store the intermediate results. Oh! that's very bad. When I check the query plan it was fine, no warning in that. Even till that time, I was thinking I have more 1.5TB free disk space and running the vacuum and analyze properly. I never thought I'll get a situation like this. Because the query was doing a lot of hash joins and the number of rows filtered by each step it so huge. 

## How to solve this problem?: 

We can't do magic to solve this. There are various reasons why your query is behaving like this.

* Bad query plan.
* Distributing a large number of rows across the nodes.
* No vacuum.
* Statistics are out of date. 
* Poor workload management settings.
* Cluster size is not fit(its very rare).

You have to find out the actual reason for this spike and then only you can apply the right solution. 

## How to control this?

It's not possible for everyone to find the cause whenever your cluster's disk was full and do the troubleshooting or in other terms, we need a quick fix and give enough room for my upcoming data. So instead of running this query and get the status from the system table, I set a Query Monitoring Rule to `Abort` the query when its going to use more than 500GB for temp and saving the intermediate results. 

In QMR, we have a rule called `Memory to Disk (1MB Blocks)` set the value 500. It means, if the block reaches 500000(1 block =1MB), then it may consume 500GB then it should Abort the query. And most of the time even it reaches 500000 blocks, it's not meaning that we consumed 500GB. Because the data stored on each block can be less than 1MB, then the overall storage consumed by these blocks may be less than 500GB, but still, it's a huge number. I won't any queries to hurt my cluster. Just kill it. But you have to set the action based on your requirement. Think twice before setting the Abort option. Then we can get the details from the Alert log table. 

![Disk Based Queries in RedShift](/assets/Disk Based Queries in RedShift3.jpg "Disk Based Queries in RedShift"){:class="lazyload"}

## How to calculate the total disk space used by a query:

You can use the following SQL query to find out which queries used how much disk space on redshift cluster for temp and storing intermediate results.  

* From `svl_query_summary` table there is column `query_temp_blocks_to_disk` will tell you how many blocks used to store the data on disk by a query.
* From `stl_query` table `bytes` column will tell you exactly how much space consumed by this query. 
While joining these two tables you can get better visibility about the complete query. 

You can even add more columns from both tables to get more useful insights. 

{% highlight sql%}
-- Get the disk based queries information for last 2 days
SELECT q.query, 
       q.endtime - q.starttime             AS duration, 
       SUM(( bytes ) / 1024 / 1024 / 1024) AS GigaBytes, 
       aborted, 
       q.querytxt 
FROM   stl_query q 
       join svl_query_summary qs 
         ON qs.query = q.query 
WHERE  qs.is_diskbased = 't' 
       AND q.starttime BETWEEN SYSDATE - 2 AND SYSDATE 
GROUP  BY q.query, 
          q.querytxt, 
          duration, 
          aborted 
ORDER  BY gigabytes DESC 
{% endhighlight %}
