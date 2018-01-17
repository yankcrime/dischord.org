---
layout: post
title: Cleaning up failed Nova instance migrations
date: 2016-09-21 14:16
published: true
categories: OpenStack
---

A quick one from my notes.  If you're blighted with logging spam from `nova-compute` along the lines of:

```
Migration instance not found: Instance b4ff3ef3-3cb2-485a-b93c-f23d6dfc91ce could not be found.
```

Then you've some failed instance migration metadata lingering in your Nova database that needs to be purged.  Fortunately this is easy enough to fix, as long as you're happy with manually deleting data that is 😉 

First up, find the corresponding row from the `migrations` table in the Nova DB:

```
MariaDB [nova]> select id,status,instance_uuid,source_compute,dest_compute from migrations where instance_uuid = '0db0e707-5e29-4da4-8e23-1c5cdf9a69f7';
+-----+-----------+--------------------------------------+----------------+--------------+
| id  | status    | instance_uuid                        | source_compute | dest_compute |
+-----+-----------+--------------------------------------+----------------+--------------+
| 998 | confirmed | 0db0e707-5e29-4da4-8e23-1c5cdf9a69f7 | alabama        | agenda       |
+-----+-----------+--------------------------------------+----------------+--------------+
1 row in set (0.00 sec)
```

Then it's simply a case of removing the offending entry:

```
MariaDB [nova]> delete from migrations where id = '998' limit 1;
Query OK, 1 row affected (0.00 sec)

MariaDB [nova]> select id,status,instance_uuid,source_compute,dest_compute from migrations where instance_uuid = '0db0e707-5e29-4da4-8e23-1c5cdf9a69f7';
Empty set (0.00 sec)
```

> NB: You don't have to restart any services; You should find that once `nova-compute` synchronises its state over the next few minutes then that spam ceases.

