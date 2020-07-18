---
layout: post
date: 2020-07-18 13:20:00 +0530
title: AWS RDS pg_dump ERROR LOCK TABLE IN ACCESS SHARE MODE for rds_superuser
description: AWS RDS PostgreSQL 9.6, pg_dump will throw the error LOCK TABLE IN ACCESS SHARE MODE even for RDS master user or rds_superuser. Here we explain how to solve this issue.
categories:
- Postgresql
tags:
- aws
- postgresql
- rds
- backup and recovery
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/pg_dump-error/aws-rds-pg_dump-error-lock-table-access-share-mode.jpg"
---
AWS RDS has all the PostgreSQL versions. But still, a lot of companies are running their database on PostgreSQL 9.6. Recently I was working on a migration project where I need to dump and restore the PostgreSQL database to a different server. You may already hear this issue and you might be solved it. But I didn't any SO questions or any blogs. So I wanted to write one for the Non-DBAs. So they can get benefit from this. 

## RDS_SUPERUSER

In RDS we can't create/grant native postgresql's`super_user` privilege instead we can grant `rds_superuser`. But its not a super user. There is a good article from [2ndquadrant](https://www.2ndquadrant.com/en/blog/the-rds_superuser-role-isnt-that-super/). 

## Problem statement:

In AWS RDS PostgreSQL (v 9.6 or lesser than 9.6), the rds_superuser can't read/write the on the tables which are owned by a different user. To perform read and write the table owner needs to grant the privileage to the rds_superuser's speratly. This is fully right? 

A lot of people already aware of this, but how about pg_dump? thats what we are going to reproduce.

## Reproduce the issue:

I have spin up an RDS PostgreSQL 9.6 to reproduce this issue.
RDS Master user name: postgres
Password: postgres

```sql
psql -h psql -h bhuvi-pg-96.chbcar19iy5o.us-east-1.rds.amazonaws.com -U postgres -d testdb

testdb=> create user appuser with password 'apppassword';
testdb=> grant CONNECT on DATABASE testdb to appuser ;
testdb=> grant all on SCHEMA public to appuser ;

-- exit
testdb=>\q 
```
Now login to the DB server with the `appuser` and create a sample table.
```sql
psql -h bhuvi-pg-96.chbcar19iy5o.us-east-1.rds.amazonaws.com -U appuser -d testdb

testdb=> create table numbers (num int);
testdb=> insert into numbers values (1),(2);

```
Lets take the dump with the master user(part of rds_superuser role)
```sql
pg_dump -h bhuvi-pg-96.chbcar19iy5o.us-east-1.rds.amazonaws.com -U postgres -d testdb > dump.sql
pg_dump: error: query failed: ERROR:  permission denied for relation numbers
pg_dump: error: query was: LOCK TABLE public.numbers IN ACCESS SHARE MODE
```
## What is the Root cause?

As I explained from the beginning, the RDS master user has the `rds_superuser` role, but its not the real super user. So it cannot read or write on the tables that are owned by a different user. 

## Solution: 

To solve this issue, the table owner must be grant the select(and other access) to the RDS master user. Then we can take the dump.

```sql
psql -h bhuvi-pg-96.chbcar19iy5o.us-east-1.rds.amazonaws.com -U appuser -d testdb
grant SELECT  on public.numbers to postgres ;

-- Now take the dump with Postgres user:
 pg_dump -h bhuvi-pg-96.chbcar19iy5o.us-east-1.rds.amazonaws.com -U postgres -d testdb > dump.sql
```
If you have a tons of tables in your database, it may difficult to do the grant one by one. So you can use the following script to generate the grant staement and execute.

```bash
#/bin/bash

users='appuser analytics dba devops'
rds_endpoint='your-rds-endpoint'
db_name='testdb'
dump_user='your-dump-user-name'

for user in $users
do
psql -h $rds_endpoint -d $db_name -U $user -t -c"select 'grant select on '||schemaname||'.'||tablename||' to $dump_user' from pg_tables where tableowner=current_user;" > /tmp/$user-grants.sql

psql -h $rds_endpoint -d $db_name -U $user  < /tmp/$user-grants.sql
done
```


## Conclusion:

We found this issue on RDS PostgreSQL 9.6 and lesser than that. But it is fixed on 10,11 and 12. Even though it is very older version, but may companies are still running this. So it would be great that AWS fix it from their end.