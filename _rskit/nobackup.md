---
layout: post
date: 2020-06-12 22:00:00 +0530
title: Tables without backup
description: RedShift toolkit result - Tables without backup
categories:
- RedShift
tags:
- aws
- redshift
- rstoolkit
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/RedShift Unload All Tables To S3.jpg"
permalink: /rskit/nobackup
---

## RStoolKit Result: Tables without backup

RedShift will take the snapshot automatically at every 8 hours or every 5GB data changes. Also whenever you need you can take manual snapshot as well. But while creating the table if you mentioned `**[BACKUP NO](https://docs.aws.amazon.com/redshift/latest/dg/r_CREATE_TABLE_NEW.html)**` in that table will include in the snapshot. By default this option is YES, but make sure that you have this `NO BACKUP` flag only for test or temporary purpose tables. 

## Find tables without backup:

```sql
select
	b."schema",
	a.name
FROM
	stv_tbl_perm a
join svv_table_info b on
	a.id = b.table_id
WHERE
	BACKUP = 0
	and temp = 0;
```

## How to fix this problem:

If you are sure that the table list returned by the above query is not need to backup, then its fine. Else we can't alter the table to enable it. You need to create a new table and copy this data.

## External Links:

1. [Create Table in RedShift](https://docs.aws.amazon.com/redshift/latest/dg/r_CREATE_TABLE_NEW.html)
2. [RedShift snapshots](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-snapshots.html)