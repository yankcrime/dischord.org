---
layout: post
title: Archiving data in Nova's database
date: 2015-12-30 16:59
published: true
categories: openstack
---

> Update:  Good news on the below.  The issue with `nova-manage db archive_deleted_rows` has been [fixed in Newton](https://review.openstack.org/#/c/299474/), and also [backported to Mitaka](https://review.openstack.org/#/c/326730/).  So if you're running a fairly recent installation of Nova then you've a properly supported option for archiving this data prior to deletion.  There's also [this blueprint](https://blueprints.launchpad.net/nova/+spec/purge-deleted-instances-cmd) which would handle purging the data entirely, hopefully slated for implementation in Ocata.

OpenStack Nova's database can grow to a significant size over time, thanks to the fact that entries
in the `instances` table (amongst others) aren't actually deleted, they're only flagged as such:

	MariaDB [nova]> select count(*) from instances;
	+----------+
	| count(*) |
	+----------+
	|    66746 |
	+----------+
	1 row in set (0.02 sec)

	MariaDB [nova]> select count(*) from instances where deleted_at is not null;
	+----------+
	| count(*) |
	+----------+
	|    66344 |
	+----------+
	1 row in set (0.26 sec)

And of course it doesn't take long for a reasonably busy platform to build up like that.  The
knock-on effect is that certain API calls ending up being particularly slow, notably
`os-simple-tenant-usage`, as described in this [bug
here](https://bugs.launchpad.net/nova/+bug/1421471).  This API is also what's used when you login to
Horizon with an account that has the 'admin' role assigned, and unfortunately it means that it can
take a long time to login as more and more cruft accumulates in the database.

Until a while back this was an easy problem to stay on top of using the `nova-manage` command, but
sadly this functionality broke ["at some point" because of a change that was
introduced](http://lists.openstack.org/pipermail/openstack-dev/2014-December/052896.html).  There's
work being done to properly address this but it doesn't look like it'll land until the next release.

Fortunately in the meantime someone has committed a couple of scripts to the
[OSOps](https://wiki.openstack.org/wiki/Osops) repo that takes
care of handling this archival process using Percona's
[pt-archiver](https://www.percona.com/doc/percona-toolkit/2.1/pt-archiver.html) tool and it does the
job nicely.  For example, you can obtain a preview of what needs to be done using
`openstack_db_archive_progress.sh`:

    # ./openstack_db_archive_progress.sh -d nova -H localhost -u root -p nova123
    Wed Dec 30 18:52:24 GMT 2015 nova.block_device_mapping has 1048, 67306 ready for archiving
	and 0 already in shadow_block_device_mapping. Total records is 68354
	Wed Dec 30 18:52:24 GMT 2015 nova.instance_metadata has 75, 75119 ready for archiving and 0
	already in shadow_instance_metadata. Total records is 75194
	Wed Dec 30 18:52:25 GMT 2015 nova.instance_system_metadata has 1187555, 50555 ready for
	archiving and 0 already in shadow_instance_system_metadata. Total records is 1238110
	Wed Dec 30 18:52:26 GMT 2015 nova.instance_actions has 150350, 0 ready for archiving and 0
	already in shadow_instance_actions. Total records is 150350
	Wed Dec 30 18:52:26 GMT 2015 nova.instance_faults has 1619, 15409 ready for archiving and 0
	already in shadow_instance_faults. Total records is 17028
	Wed Dec 30 18:52:26 GMT 2015 nova.virtual_interfaces has 0, 0 ready for archiving and 0
	already in shadow_virtual_interfaces. Total records is 0
	Wed Dec 30 18:52:26 GMT 2015 nova.fixed_ips has 0, 0 ready for archiving and 0 already in
	shadow_fixed_ips. Total records is 0
	Wed Dec 30 18:52:26 GMT 2015 nova.security_group_instance_association has 0, 0 ready for
	archiving and 0 already in shadow_security_group_instance_association. Total records is 0
	Wed Dec 30 18:52:26 GMT 2015 nova.migrations has 218, 0 ready for archiving and 0 already in
	shadow_migrations. Total records is 218
	Wed Dec 30 18:52:26 GMT 2015 nova.instance_extra has 358, 51401 ready for archiving and 0
	already in shadow_instance_extra. Total records is 51759

And then to actually do the archival it's simply a case of running:

	# ./openstack_db_archive.sh -d nova -H localhost -u root -p nova123

By way of comparison, here's how long `nova usage-list` (which hits the `os-simple-tenant-usage` endpoint) took before:

	# time nova usage-list
	[..]
	real    0m45.250s
	user    0m0.592s
	sys    0m0.252s

And after:

	# time nova usage-list
	[..]
	real    0m24.126s
	user    0m0.355s
	sys    0m0.109s

Timings and data are from a development environment in VMs (running OpenStack Kilo, with Galera / MariaDB) on my
laptop, so take from that what you will!

Finally, the scripts themselves are mirrored on GitHub here:
[https://github.com/openstack/osops-tools-generic/tree/master/nova](https://github.com/openstack/osops-tools-generic/tree/master/nova).
