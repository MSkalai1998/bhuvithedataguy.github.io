---
layout: post
date: 2020-09-28 15:50:00 +0530
title: Access CloudSQL PostgreSQL FDW From Different VPC Or Public 
description: Access GCP CloudSQL instances from different VPC or different projects or even through public IP via PostgreSQL FDW.
categories:
- GCP
tags:
- gcp
- cloudsql
- dba
- postgres
- postgres_fdw
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/postgres/Access CloudSQL PostgreSQL FDW From Different VPC Or Public2.jpg"
---
Access GCP CloudSQL instances from different VPC or different projects or even through public IP via PostgreSQL FDW(foreign data wrapper).

GCP CloudSQL Postgres started supporting to install Postgres FDW. They introduced this feature after a long time. Its a cool extension to easily query the different databases from the same server or access some small tables(Like lookup tables) on the analytics server. But in CloudSQL there is a glitch. We can't use this extension outside the VPC.(at the time when I writing this post). See the [documentation](https://cloud.google.com/sql/docs/postgres/extensions#postgres_fdw).

> Currently, this extension works for two Cloud SQL private IP instances within the same VPC network, or for cross databases within the same instance.

Recently I had a requirement that we want to access some of the production database tables from a different project(let's call this project as the data engg project).  So I created the FDW using the Public IP of my Production CloudSQL instance. Initially, I didn't notice this Private communication limitation, and this approach was failed. Later I read the doc properly. 

## Bypassing Private communication:

It is mentioned as the communication will happen within the VPC. In GCP if you peered the CloudSQL with your VPC, by default all the IP addresses from your VPC will be whitelisted to access the CloudSQL. So I wanted to try a small hack with a proxy. 

{% include lazyload.html image_src="/assets/bq-cf/postgres/Access CloudSQL PostgreSQL FDW From Different VPC Or Public1.jpg" image_alt="postgres/Access CloudSQL PostgreSQL FDW From Different VPC Or Public" image_title="postgres/Access CloudSQL PostgreSQL FDW From Different VPC Or Public" %}

* Spin up a new VM on the analytics project and attach an external IP.
* Whitelist the analtics Proxy VM's Public IP on the Production CloudSQL instance's connections.
* Create a port forwarding rule from the Proxy VM to the Production project CloudSQL. 
* Create a firewall rule for the Proxy VM to allow the proxy port from all the IP that are allocated for `Private Secure Connection`. You can get the IP ranges from the VPC console and see the connection name as `cloudsql-postgres-googleapis-com`(Basically Analytic CloudSQL FDW will talk to Proxy VM, so it should accpt the incoming connections from the CloudSQL. I tried to whitelist the private IP of CloudSQL, it didn't work, so we have to whitelist the whole Private IP range)
* Create a Postgres FDW on the Analytics CloudSQL with Proxy VM's Private IP.
* Create user mapping and the tables.

## Steps to setup the FDW between the different projects.

Create the user on Production CloudSQL with Read Only access.
```sql
CREATE USER fdw_read with password 'strong-pass';
GRANT SELECT on ALL  tables in schema myschema to fdw_read;
```
Create a table on Prod CloudSQL and insert some rows.
```sql
CREATE TABLE myschema.test (
id int
);
INSERT INTO myschema.test values (1);
```
**Follow the below steps on the Analytic CloudSQL**
Create the port forwarding with SOCAT. (assume `30.30.30.30` is the public IP of my production cloudsql. And make sure that this `socat` should be running always. So use `supervisord` or run this as a `systemctl` service. 
```bash
apt install socat
socat TCP-LISTEN:5434,fork TCP:30.30.30.30:5432 
```
Create a firewall rule to allow 5434 port from the IP range of Private secure connection. Connection name `cloudsql-postgres-googleapis-com`.
{% include lazyload.html image_src="/assets/bq-cf/postgres/Access CloudSQL PostgreSQL FDW From Different VPC Or Public3.jpg" image_alt="postgres/Access CloudSQL PostgreSQL FDW From Different VPC Or Public" image_title="postgres/Access CloudSQL PostgreSQL FDW From Different VPC Or Public" %}

Install the FDW extension on the analytic database.
```sql
CREATE EXTENSION postgres_fdw;
```
I have a non-super user for running analytic queries. So we need to grant the usage permission for the FDW extension and the FDW server. (for the super user's it's not required)
```sql
GRANT USAGE ON FOREIGN DATA WRAPPER postgres_fdw to dataengg_read;
GRANT USAGE ON FOREIGN SERVER prod_pg TO dataengg_read;
```
Create the server definition. (assume `10.0.1.10` is the private IP of my proxy VM)
```sql
CREATE SERVER prod_pg 
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (dbname 'proddb', host '10.0.1.10', port '5434');
```
Map an existing user(dataengg_read) to Prod CloudSQL's FDW user.
```sql
CREATE USER MAPPING for dataengg_read
SERVER prod_pg
OPTIONS (user 'fdw_read ', password 'strong-pass');
```
Create the foreign table.
```sql
CREATE FOREIGN TABLE fdtest
(id int)
SERVER prod_pg
OPTIONS (schema_name 'myschema', table_name 'test');
```
### Testing

Now, lets access the data from the Analytics database.
```sql
SELECT * FROM fdtest;
 id 
----
  1
```
Update the row on the production and see the updated row on the analytic server.
```sql
production=>update fdw_test set id =4;
analytics=> select * from fdtest;
 id 
----
  4
```
## Recommended approach for more secure access

{% include lazyload.html image_src="/assets/bq-cf/postgres/Access CloudSQL PostgreSQL FDW From Different VPC Or Public2.jpg" image_alt="postgres/Access CloudSQL PostgreSQL FDW From Different VPC Or Public" image_title="postgres/Access CloudSQL PostgreSQL FDW From Different VPC Or Public" %}

1. In my case, we enabled the public IP connection on the production DB(for some third party API), but if you want to keep everything in private then follow this approach.
2. Setup VPC peering between your source and destination projects.
3. Create Proxy VM on the source project.
4. Create the port forwarding from source proxy VM to source cloudsql.
5. Create Proxy VM on the destination project.
6. Create a firewall rule for destination proxy vm to allow proxy port from the whole IP ranges of Private Secure connection(you can get it from the VPC console)
7. Create the port forwarding from destination proxy VM to source proxy VM.
8. Create the foreign server on the destination cloudsql with the IP of the destination project's proxy VM.


