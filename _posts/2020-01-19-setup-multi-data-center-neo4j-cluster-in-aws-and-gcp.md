---
title: Setup Multi Data Center Neo4j Cluster In AWS and GCP
date: 2020-01-19 23:15:00 +0530
description: 'In this blog we covered how to setup a multi datacenter or multi region
  neo4j cluster on AWS and GCP. '
categories:
- Neo4j
tags:
- Neo4j
- Graph-Database
- Cluster
- Replication
image: "/assets/Setup Multi Data Center Neo4j Cluster In AWS and GCP.jpg"

---
Neo4j's multi datacenter deployments are well suited for a geo-distributed workload and also provide a better disaster recovery solution. But to be frank, its not an actual distributed databases like Google Spanner or CocroachDB. Here it's just grouping/labeling your Neo4j Nodes with different data center names. Even though it has a lot more benefits, like load balancing to a particular group, replicating the data to read replica from the existing read replica instead of replicating from master and etc. Like my [**previous blog**](https://thedataguy.in/setup-neo4j-causal-cluster-on-gcp-and-aws/), this also just guides to setting up the Multi datacenter cluster in AWS and GCP.

## AWS/GCP:

This blog just gives you simple steps to create a fresh Neo4j Multi data center cluster. From AWS/GCP you just need to whitelist the IP address in the security group(AWS) and Firewall rules(GCP). Otherwise, all the steps are common for both deployments.

## Setup Details:

| Name | IP | Region | Role |
| --- | --- | --- | --- |
| Node-1 | 10.128.0.98 | us-central | CORE |
| Node-2 | 10.142.0.4 | us-east-1 | CORE |
| Node-3 | 10.128.0.102 | us-central | REPLICA |
| Node-4 | 10.128.0.99 | us-central | REPLICA |
| Node-5 | 10.142.0.23 | us-east-1 | REPLICA |
| Node-6 | 10.142.0.16 | us-east-1 | REPLICA |

## Install Neo4j on all the nodes:

{% highlight shell %}
apt-get -y update
apt -y install openjdk-8-jre
wget -O - https://debian.neo4j.org/neotechnology.gpg.key | sudo apt-key add -
echo 'deb https://debian.neo4j.org/repo stable/' | sudo tee -a /etc/apt/sources.list.d/neo4j.list
sudo apt-get -y update
sudo apt-get -y install neo4j-enterprise=1:3.5.14
neo4j-admin set-initial-password 'root'
{% endhighlight %}

## Necessary Ports:

1. 5000 - discovery_listen_address
2. 6000 - transaction_advertised_address
3. 7000 - raft_advertised_address
4. 7473 - HTTPS interface to access the Neo4j cluster in browser
5. 7474 - HTTP  interface to access the Neo4j cluster in browser
6. 7687 - Used by Cypher Shell and by Neo4j Browser
7. 6362 - Backup port to seed the data from the Leader node.

Please allow the above ports between all the nodes.

## Configure Multi Datacenter Cluster:

In our setup, we use 2 nodes as a minimum number of nodes to form a cluster, also we always need 2 runtime nodes to make the cluster up and running. Update the following values in the `/etc/neo4j/neo4j.conf` file. Before doing this just stop the neo4j service and delete the store data (`/var/lib/neo4j/data/databases/graph.db/`)

{% highlight shell %}
Node 1
dbms.security.auth_enabled=true
dbms.connectors.default_listen_address=0.0.0.0
dbms.connectors.default_advertised_address=10.128.0.98
dbms.mode=CORE
causal_clustering.minimum_core_cluster_size_at_formation=3
causal_clustering.minimum_core_cluster_size_at_runtime=3
causal_clustering.initial_discovery_members=10.128.0.98:5000,10.142.0.4:5000
causal_clustering.initial_discovery_members=10.128.0.98:5000,10.142.0.4:5000
causal_clustering.discovery_listen_address=0.0.0.0:5000
causal_clustering.raft_advertised_address=10.128.0.98:7000
causal_clustering.transaction_advertised_address=10.128.0.98:6000
causal_clustering.server_groups=us-central-1

Node 2
dbms.security.auth_enabled=true
dbms.connectors.default_listen_address=0.0.0.0
dbms.connectors.default_advertised_address=10.142.0.4
dbms.mode=CORE
causal_clustering.minimum_core_cluster_size_at_formation=2
causal_clustering.minimum_core_cluster_size_at_runtime=2
causal_clustering.initial_discovery_members=10.128.0.98:5000,10.142.0.4:5000
causal_clustering.initial_discovery_members=10.128.0.98:5000,10.142.0.4:5000
causal_clustering.discovery_listen_address=0.0.0.0:5000
causal_clustering.raft_advertised_address=10.142.0.4:7000
causal_clustering.transaction_advertised_address=10.142.0.4:6000
causal_clustering.server_groups=us-east-1
{% endhighlight %}

## Start the Neo4j

Now we can start the neo4j service it'll form a 2 node cluster.
{% highlight shell %}
service neo4j start
{% endhighlight %}

{% highlight sql %}
neo4j> CALL dbms.cluster.overview();

| id                                     | addresses                                                                          | role       | groups           | database  |

| "e1ee4fd2-6698-470a-950c-a143bab5d904" | \["bolt://10.128.0.98:7687", "http://10.128.0.98:7474", "https://10.128.0.98:7473"\] | "LEADER"   | \["us-central-1"\] | "default" |
| "a19a9bc8-ad89-4451-bbd1-8d5f35190eda" | \["bolt://10.142.0.4:7687", "http://10.142.0.4:7474", "https://10.142.0.4:7473"\]    | "FOLLOWER" | \["us-east-1"\]    | "default" |

{% endhighlight %}

## Adding the Replica (us-central)

Edit the `neo4j.conf` file on the node 3 and 4.

{% highlight shell %}
Node 3
dbms.security.auth_enabled=true
dbms.connectors.default_listen_address=0.0.0.0
dbms.connectors.default_advertised_address=10.128.0.99
dbms.mode=READ_REPLICA
causal_clustering.minimum_core_cluster_size_at_formation=3
causal_clustering.minimum_core_cluster_size_at_runtime=3
causal_clustering.initial_discovery_members=10.128.0.98:5000
causal_clustering.discovery_listen_address=0.0.0.0:5000
causal_clustering.raft_advertised_address=10.128.0.99:7000
causal_clustering.transaction_advertised_address=10.128.0.99:6000
causal_clustering.server_groups=us-central-1

Node 4
dbms.security.auth_enabled=true
dbms.connectors.default_listen_address=0.0.0.0
dbms.connectors.default_advertised_address=10.128.0.102
dbms.mode=READ_REPLICA
causal_clustering.minimum_core_cluster_size_at_formation=3
causal_clustering.minimum_core_cluster_size_at_runtime=3
causal_clustering.initial_discovery_members=10.128.0.98:5000
causal_clustering.discovery_listen_address=0.0.0.0:5000
causal_clustering.raft_advertised_address=10.128.0.102:7000
causal_clustering.transaction_advertised_address=10.128.0.102:6000
causal_clustering.server_groups=us-central-1
{% endhighlight %}

## Adding the Replica (us-east-1)

Edit the `neo4j.conf` file on the node 5 and 6.

{% highlight shell %}
Node 5
dbms.security.auth_enabled=true
dbms.connectors.default_listen_address=0.0.0.0
dbms.connectors.default_advertised_address=10.142.0.23
dbms.mode=READ_REPLICA
causal_clustering.minimum_core_cluster_size_at_formation=2
causal_clustering.minimum_core_cluster_size_at_runtime=2
causal_clustering.initial_discovery_members=10.142.0.4:5000
causal_clustering.discovery_listen_address=0.0.0.0:5000
causal_clustering.raft_advertised_address=10.142.0.23:7000
causal_clustering.transaction_advertised_address=10.142.0.23:6000
causal_clustering.server_groups=us-east-1

Node 6
dbms.security.auth_enabled=true
dbms.connectors.default_listen_address=0.0.0.0
dbms.connectors.default_advertised_address=10.142.0.16
dbms.mode=READ_REPLICA
causal_clustering.minimum_core_cluster_size_at_formation=2
causal_clustering.minimum_core_cluster_size_at_runtime=2
causal_clustering.initial_discovery_members=10.142.0.4:5000
causal_clustering.discovery_listen_address=0.0.0.0:5000
causal_clustering.raft_advertised_address=10.142.0.16:7000
causal_clustering.transaction_advertised_address=10.142.0.16:6000
causal_clustering.server_groups=us-east-1
{% endhighlight %}

## Start the Replica nodes:

Now start the neo4j service on all the read replica nodes, and then check the cluster status.
{% highlight shell %}
service neo4j start
{% endhighlight %}

{% highlight sql %}
neo4j> CALL dbms.cluster.overview();
| id                                     | addresses                                                                             | role           | groups           | database  |
\+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| "74124822-6963-49a5-af49-2c69132bfa0b" | \["bolt://10.128.0.102:7687", "http://10.128.0.102:7474", "https://10.128.0.102:7473"\] | "READ_REPLICA" | \["us-central-1"\] | "default" |
| "e1ee4fd2-6698-470a-950c-a143bab5d904" | \["bolt://10.128.0.98:7687", "http://10.128.0.98:7474", "https://10.128.0.98:7473"\]    | "LEADER"       | \["us-central-1"\] | "default" |
| "bb1c383e-39ff-46a8-9a03-3df2d5967029" | \["bolt://10.128.0.99:7687", "http://10.128.0.99:7474", "https://10.128.0.99:7473"\]    | "READ_REPLICA" | \["us-central-1"\] | "default" |
| "f7dae729-cf87-4c62-a68f-1fa067b9a3fa" | \["bolt://10.142.0.16:7687", "http://10.142.0.16:7474", "https://10.142.0.16:7473"\]    | "READ_REPLICA" | \["us-east-1"\]    | "default" |
| "3199a28f-f7af-49a4-adcb-918aab086eb8" | \["bolt://10.142.0.23:7687", "http://10.142.0.23:7474", "https://10.142.0.23:7473"\]    | "READ_REPLICA" | \["us-east-1"\]    | "default" |
| "a19a9bc8-ad89-4451-bbd1-8d5f35190eda" | \["bolt://10.142.0.4:7687", "http://10.142.0.4:7474", "https://10.142.0.4:7473"\]       | "FOLLOWER"     | \["us-east-1"\]    | "default" |
{% endhighlight %}

Our multi datacenter cluster is ready. It just gives you a simple configuration guide for multi data center design. But I didn't cover the best practices here. May be I'll write a new blog for that.

## References:

1. [Multi Datacenter Design samples](https://neo4j.com/docs/operations-manual/3.5/clustering-advanced/multi-data-center/design/#multi-dc-core-server-deployment-scenarios)
2. [More features for using Multi datacenter design](https://neo4j.com/docs/operations-manual/3.5/clustering-advanced/multi-data-center/configuration/)
3. [Load balancing with Multi datacenter](https://neo4j.com/docs/operations-manual/3.5/clustering-advanced/multi-data-center/load-balancing/)