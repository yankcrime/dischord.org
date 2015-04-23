---
layout: post
title: Out-of-sync quotas in OpenStack Nova
date: 2015-04-17 14:54
published: true
categories:
---

From time-to-time user quotas in OpenStack become out of sync, i.e usage will show X number of instances in use and the reality, Y, is a different value altogether.  Until now the way to fix this has been to manually update the `nova` database and amend the `quota_usages` table and then trigger a refresh by launching a new machine, i.e:

```
update quota_usages set in_use = 0 where id = 890 and project_id = '509d6ae853114ea9aaaac02804d3a4dd' limit 1;
```

Or potentially updating en masse - if you're feeling brave - with something like:

```
update quota_usages, (select usage_id, sum(delta) as sum from reservations where project_id='86d829ffc0ff4efb943131f7d2a18d52' and deleted!=id group by usage_id) as r set quota_usages.in_use = r.sum where quota_usages.id = r.usage_id;
update quota_usages, (select distinct(usage_id) as usage_id from reservations where project_id='ad3e3ee7a08d45df965908704f29b873' and deleted=id ) as r set quota_usages.in_use = 0 where quota_usages.id = r.usage_id;
```

However, the CERN chaps have just released a handy little tool that checks and then updates where necessary if there's any mismatches.  They've blogged a bit about it [here](http://openstack-in-production.blogspot.co.uk/2015/03/nova-quota-usage-synchronization.html) and the script itself is over [here](https://github.com/cernops/nova-quota-sync).
