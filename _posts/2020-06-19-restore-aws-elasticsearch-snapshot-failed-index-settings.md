---
title: Restore AWS ElasticSearch Snapshot Failed - Index settings
date: 2020-06-19 13:00:00 +0530
description: Restoring an AWS managed elasticsearch cluster snapshot is failed with node does not match index setting index.routing.allocation.include and the filter is instance_type i2.2xlarge OR i3.2xlarge. 
categories:
- ElasticSearch
tags:
- aws
- elasticsearch
image: "/assets/Restore AWS ElasticSearch Snapshot Failed/Restore AWS ElasticSearch Snapshot Failed - Index settings.jpg"
---
AWS Managed ElasticSearch provides the flexibility to export the snapshots to the S3 bucket. We can restore them anywhere. Generally, it is very straight forward. Create a repository and just restore the indices. But today I found a strange issue while restoring a managed ElasticSearch's snapshot to a VM. The interesting thing is it never produces any error in the log file. The moment when I enter the restore command, immediately the cluster status went to RED. 

The source ElasticSearch cluster version is 5.6. We took a snapshot of 2 indices. On the VM we installed the ElastcSearch software and created the snapshot repo.

```bash
curl -X POST localhost:9200/_snapshot/awsrestore/snap_id/_restore 
```
Then I looked at the status of the restore.

```bash
curl -X GET localhost:9200/_recovery?pretty=true
{}
```
It is empty. Then I checked the cluster status, it was Red.

I checked the logs on the master nodes and the data nodes, but nothing was there. Then I started thinking will troubleshoot step by step. 

# #1: I don't have any clue

The Log file says, everything is fine, No reason why it's failing. I have only one thing in which the state is Red. Fine, will see what else we can get from the status.
```bash
curl -X GET http://localhost:9200/_cluster/health?pretty=true

{
  "cluster_name" : "es-cluster",
  "status" : "red",
  "timed_out" : false,
  "number_of_nodes" : 8,
  "number_of_data_nodes" : 5,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 480,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 0.0
}
```
Apart from status, the other fishy thing is `unassigned_shards` is 480. Lets verify this.

```bash
curl -XGET localhost:9200/_cat/shards?h=index,shard,prirep,state,unassigned.reason| grep UNASSIGNED


my_index_name 6   p UNASSIGNED   NEW_INDEX_RESTORED
my_index_name 9   r UNASSIGNED   NEW_INDEX_RESTORED
my_index_name 116 r UNASSIGNED   NEW_INDEX_RESTORED
my_index_name 73  p UNASSIGNED   NEW_INDEX_RESTORED
my_index_name 90  r UNASSIGNED   NEW_INDEX_RESTORED
my_index_name 50  r UNASSIGNED   NEW_INDEX_RESTORED
my_index_name 23  p UNASSIGNED   NEW_INDEX_RESTORED
```

# #2 why it is unassigned: 

Now we have a clue that all the shards in the index are unassigned. Lets extract the reason why it is unassigned. 
``bash
curl -XGET localhost:9200/_cluster/allocation/explain?pretty

--Output truncated

"node_allocation_decisions" : [
    {
      "node_id" : "3WEV1tHoRPm6OguKyxp0zg",
      "node_name" : "node-5",
      "transport_address" : "10.10.10.9:9300",
      "node_decision" : "no",
      "deciders" : [
        {
          "decider" : "replica_after_primary_active",
          "decision" : "NO",
          "explanation" : "primary shard for this replica is not yet active"
        },
        {
          "decider" : "filter",
          "decision" : "NO",
          "explanation" : "node does not match index setting [index.routing.allocation.include] filters [instance_type:\"i2.2xlarge OR i3.2xlarge\"]"
        },
        {
          "decider" : "throttling",
          "decision" : "NO",
          "explanation" : "primary shard for this replica is not yet active"
        }
      ]
    }
```

Take a look at the output on each node. They all are saying the same reason 
1. primary shard for this replica is not yet active
2. node does not match index setting
3. primary shard for this replica is not yet active.
 
Ser the second error message. This error is something new, you may get the same error, but the see the filer. It is indicating something `i2.2xlarge OR i3.2xlarge` which is uncommon. But we know that this is our Managed ElasticSearch instance type. So now we found that the indices are having an AWS specific filter. 

# #3 How to remove this filter:

We can skip this filter while taking the snapshot, but in my case, its already done and its pretty huge snapshot (a few TBs). So I need to look at the alternate options. I was thinking, is this possible to skip this particular setting while restoring?. 

Yes, it is possible. Here is the [documentation link](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-restore-snapshot.html#_changing_index_settings_during_restore). 

# #4 Restore the snapshot:

We are good now, we found the cause and the solution. But what is the exact setting name that we should use to skip?. 
We need to get that from the source cluster or even the previous restoration failed but the index got created without the data. Lets get the setting from the target cluster itself. Or you can simply get it from the allocation API output as well.

```json
curl -X GET localhost:9200/my_index_name/_settings?pretty

{
  "my_index_name" : {
    "settings" : {
      "index" : {
        "codec" : "best_compression",
        "routing" : {
          "allocation" : {
          "include": {
          "instance_type": "i2.2xlarge OR i3.2xlarge"
          }
            "exclude" : {
              "layer" : "performance"
            }
          }
        },
```
Now, restore the snapshot with the following command.
```bash
curl -X POST "localhost:9200/_snapshot/awsrestore/gcp-testing-2020-06-09/_restore?pretty" -H 'Content-Type: application/json' -d' {"ignore_index_settings": ["index.routing.allocation.include.instance_type"]}'
```
It worked, this time the cluster didn't go to Red. And the restoration process is started. I didn't find any blog or forum questions. Even I raise a question in [StackOverflow](https://stackoverflow.com/questions/62469731/aws-managed-elastic-search-restore-node-does-not-match-index-setting). So I thought to write about it.