---
title: Why RedShift is showing rows_pre_user_filter is zero
date: 2020-06-08 20:20:00 +0530
description: If the RedShift is using late materialization, then from the STL_SCAN table the rows_pre_user_filter will show the value 0.
categories:
- RedShift
tags:
- aws
- redshift
image: "/assets/redshift-row-pre-user-filter-zero/Why RedShift is showing rows_pre_user_filter is zero.jpg"
---
I was working for a script to figure out the Ghost rows in all the tables based on the `STL_SCAN`. This system table has a detailed view of your query execution. The column `rows_pre_filter` will tell you how many rows fetched from the disk including the rows that are marked for delete(Ghost rows). Then the next step is to filter that row offset by removing the Ghost rows. Then it'll apply the user filters like where conditions. I found strange behavior in this `STL_SCAN` table. 

Let's say you have a table with 10 rows. Then you delete the 5 rows. 
{% highlight sql%}
create table grows_test (id int);

insert into grows_test values (1);
insert into grows_test values (2);
insert into grows_test values (3);
insert into grows_test values (4);
insert into grows_test values (5);
insert into grows_test values (6);
insert into grows_test values (7);
insert into grows_test values (8);
insert into grows_test values (9);
insert into grows_test values (10);

select * from grows_test;
{% endhighlight %}

Now get the query ID for the last executed select query.

{% highlight sql%}
select pg_last_query_id();
1274250
{% endhighlight %}

Let's see its execution from the `STL_SCAN`.
{% highlight sql%}
select
    query,
    segment,
    step,
    rows,
    rows_pre_filter,
    rows_pre_user_filter,
    perm_table_name
from
    STL_SCAN
where
    query = 1274250;
{% endhighlight %}

{% include lazyload.html image_src="/assets/redshift-row-pre-user-filter-zero/Why RedShift is showing rows_pre_user_filter is zero1.jpg" image_alt="Why RedShift is showing rows_pre_user_filter is zero" image_title="Why RedShift is showing rows_pre_user_filter is zero" %}

RedShift fetches 10 rows from the disk. After removing the deleted rows it returned 10 rows(since we didn't delete any rows). Let's delete some rows and run the select query again.
{% highlight sql%}
delete from grows_test where id > 5;

select * from grows_test;
{% endhighlight %}

This time see the query execution from the `STL_SCAN`. Get the last executed Query ID.
{% highlight sql%}
select pg_last_query_id();
1274253

select
    query,
    segment,
    step,
    rows,
    rows_pre_filter,
    rows_pre_user_filter,
    perm_table_name
from
    STL_SCAN
where
    query = 1274253;
{% endhighlight %}

{% include lazyload.html image_src="/assets/redshift-row-pre-user-filter-zero/Why RedShift is showing rows_pre_user_filter is zero2.jpg" image_alt="Why RedShift is showing rows_pre_user_filter is zero" image_title="Why RedShift is showing rows_pre_user_filter is zero" %}

This time you can see the `rows_pre_user_filter` is 5 which is the row count after eliminating the Ghost rows. This value will still remain the same even you apply any filters. 
select * from grows_test where id=1;
If you check the `STL_SCAN`, the `rows_pre_user_filter` will be 5.

But in one of my production cluster, while checking the `STL_SCAN`, I found that the `rows_pre_user_filter` was 0. Then means after eliminating the dead rows nothing is there. That's the meaning. But it's not true, it is one of my largest tables in the cluster. It should not say zero rows. But when I see the rows column(the number of rows processed) in `STL_SCAN` it has some number. How this is possible? 

{% highlight sql%}
select
    query,
    segment,
    step,
    rows,
    slice,
    rows_pre_filter,
    rows_pre_user_filter,
    perm_table_name
from
    STL_SCAN
where
    query = '1243623'
order by
    slice
{% endhighlight %}

{% include lazyload.html image_src="/assets/redshift-row-pre-user-filter-zero/Why RedShift is showing rows_pre_user_filter is zero3.jpg" image_alt="Why RedShift is showing rows_pre_user_filter is zero" image_title="Why RedShift is showing rows_pre_user_filter is zero" %}

For all the slices it is showing zero only. Then I started looking at the other columns as well. 

## Late Materialization:

RedShift supports [late materialization from 2017 onwards](https://aws.amazon.com/about-aws/whats-new/2017/12/amazon-redshift-introduces-late-materialization-for-faster-query-processing/). This late materialization will help us to do the row-level filtering while fetching the data from the disk itself. So instead of reading the whole block, only filtered rows will be selected. This is the catch here. Redshift will not use late materialization for all the queries, but whenever it's using the late materialization for a query then from the `STL_SCAN` table it'll mark the is_rlf_scan as true. Then while checking further I noticed that if the query uses late materialization then the `rows_pre_user_filter` is zero. 

{% highlight sql%}
select
    query,
    segment,
    step,
    rows,
    slice,
    rows_pre_filter,
    rows_pre_user_filter,
    is_rlf_scan,
    perm_table_name
from
    STL_SCAN
where
    query = 1014661
order by
    slice;
{% endhighlight %}
{% include lazyload.html image_src="/assets/redshift-row-pre-user-filter-zero/Why RedShift is showing rows_pre_user_filter is zero4.jpg" image_alt="Why RedShift is showing rows_pre_user_filter is zero" image_title="Why RedShift is showing rows_pre_user_filter is zero" %}
