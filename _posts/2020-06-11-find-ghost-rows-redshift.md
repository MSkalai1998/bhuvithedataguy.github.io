---
title: Find Ghost Rows/Dead Rows For All The Tables In RedShift
date: 2020-06-11 14:10:00 +0530
description: How to find Ghost rows or dead rows in AWS RedShift with the STL SCAN table.
categories:
- RedShift
tags:
- aws
- redshift
image: "/assets/find-ghost-rows-redshift/Find Ghost Rows Dead Rows For All The Tables In RedShift.jpg"
---
Ghost rows or Dead rows in RedShift is a Red flag for the cluster's performance. RedShift is built on top of the PostgreSQL database. To support MVCC PostgreSQL will handle the delete and the updates in a different way. Delete will not remove the data from the disk/block. Instead, the row will be marked as deleted. It's a soft delete. For Updates its a combination of `Delete + Insert`. The existing row will be marked as deleted and a new row will be inserted with the updated values. In Redshift, we have 3 Pseudo columns.

1. INSERT_XID
2. DELETE_XID
3. ROW_ID (OID)

I hope the XID for the delete operation for a particular row will be added to this column. It's internal to redshift, so I can't give 100% assurance on this statement. 

The rows marked as deleted are called Dead Rows(Dead Tuples) in PostgreSQL and in RedShift, we call it as Ghost Rows. These ghost rows will be removed while running the Vacuum. In RedShift, these rows will be removed by,

* Vacuum FULL
* Vacuum Delete Only
* Auto Vacuum

`Vacuum short` will not remove the Ghost rows. Now how do we identify the Ghost rows in RedShift? Unfortunately, I try to find any system table or view, but nothing is available for now. 

## Find the Ghost Rows:

We can get the ghost rows info in two places. 

1. STL_VACUUM DETAIL - An undocumented system table
2. [STL_SCAN](https://docs.aws.amazon.com/redshift/latest/dg/r_STL_SCAN.html) - Information about query execution

From the stl_vacuum_detail  there is a column num_deleted_rows will tell you how many dead rows are removed by the vacuum process. It includes both Auto vacuum and manual vacuum details. But it's only useful after the vacuum. 

But STL_SCAN, you have two important columns.

* rows_pre_filter - Number of rows read from the block without filtering the ghost rows.
* rows_pre_user_filter - Number of rows after eliminating the ghost rows. 

It makes sense right?  The difference between the rows_pre_filter  and rows_pre_user_filter  will give you the ghost rows count. I have prepared a query to get the ghost rows info from this table. 

## The logic behind the query:

* Get the recent vacuum timestamp and the table name for the tables from STL_VACUMM. 
* STL_SCAN has a lot of queries, but we need to pick the query that ran after the vacuum for a table.
* Even if you have more queries ran on top of a table after the vacuum, then pick the latest one.
* If your query used [late materialization, then `rows_pre_user_filter` and the `rows_pre_user_filter` will be zero](https://thedataguy.in/redshift-row-pre-user-filter-zero-late-materialization/). Then it is not useful. To find the next query.
* If you use the same table twice in your query or did a self join then it may scan the table 2 times. So pick the latest segment to avoid duplicates. 
* But if your table never vacuumed or the information not available on the STL_VACUUM table, then you may miss some tables.
* So for those tables, get the recent query from the past 5 days. 
* Now do the ghost row calculation.

## Query to find the Ghost Rows:
{% highlight sql%}
with cte as (
select
	table_id,
	status,
	eventtime
from
	stl_vacuum
where
	( status = 'Started'
	or status like '%Started Delete Only%'
	or status like '%Finished%') ),
result_set as (
select
	table_id ,
	max(a.eventtime)as vacuum_timestamp
from
	cte a
where
	a.status like '%Finished%'
group by
	table_id ) ,
raw_gh as(
select
	query,
	tbl,
	perm_table_name ,
	segment,
	sum(a.rows_pre_filter) as rows_pre_filter ,
	sum(a.rows_pre_user_filter) as rows_pre_user_filter ,
	sum(a.rows_pre_filter-a.rows_pre_user_filter)as ghrows
from
	stl_scan a
LEFT JOIN result_set c on
	a.tbl = c.table_id
where
	a.starttime > coalesce(c.vacuum_timestamp, CURRENT_TIMESTAMP - INTERVAL '5 days')
	and perm_table_name not in ('Internal Worktable',
	'S3')
	and is_rlf_scan = 'f'
	and (a.rows_pre_filter <> 0
	and a.rows_pre_user_filter <> 0 )
group by
	segment,
	query,
	tbl,
	perm_table_name ),
ran as(
select
	*,
	dense_rank() over (partition by tbl
order by
	query desc,
	segment desc) as rnk
from
	raw_gh )
select
	b."schema",
	b."table",
	sum(rows_pre_filter) as total_rows,
	sum(rows_pre_user_filter) as valid_rows,
	sum(ghrows) as ghost_rows
from
	ran a
join pg_catalog.svv_table_info b on
	a.tbl = b.table_id
where
	rnk = 1 
group by
	"schema" ,
	"table";
{% endhighlight %}	

But there is one limitation here, if your table is never queried or accessed, then that table will not shown in the result.