---
layout: post
title: Nova server group affinity policy violations
date: 2016-04-11 19:17
published: true
categories: OpenStack
---

Virtual machine live migrations are a <del>crutch</del> fact of life for pretty much anyone managing a sizeable estate.  Working with [KVM](http://www.linux-kvm.org) via [OpenStack](http://www.openstack.org) Nova is no exception;  Upgrades, hardware failure, and general hypervisor maintenance often necessitate execution of `nova live-migration` to move an instance off onto another hypervisor elsewhere.  This is all well and good, and in normal operation `nova-scheduler` will handle placement where it sees fit based on the scheduler filters you've configured.

Tangentially, there's a relatively little-known (but essential) feature of Nova called [Server Groups](https://raymii.org/s/articles/Openstack_Affinity_Groups-make-sure-instances-are-on-the-same-or-a-different-hypervisor-host.html).  These allow you to - as the name suggests - group servers (instances) together and apply some kind of policy.  Mostly this is used to set affinity or anti-affinity rules, as in to make certain members of this group are instantiated on disparate hypervisors.  The use-case is obvious - you're ensuring that your failure domain for that particular group of instances is wider than a single hypervisor, something that's essential given the CPU and memory density that most cloud operators configure their compute nodes with.  Loss of a single compute node can translate to sometimes 100s of instances going offline, and if the entirety of your database server cluster sat on that one physical box then that you're well and truly in the hatezone.  Configuring a server group which contains your database cluster and setting the anti-affinity policy for that group ensures that this won't - or shouldn't - happen.

The key word there is shouldn't.  The `nova live-migration` command takes another argument, that of a target hypervisor, which an OpenStack administrator can use to arbitrarily migrate an instance to.  The problem comes about when you realise that this then basically sidesteps the `nova-scheduler` filters, including the one which enforces anti-affinity policies.  Oops.

We're still debating whether or not this is a bug (it probably is).  There was [some work done](https://review.openstack.org/#/c/135351/) a while back to ensure `live-migration` honors these affinity rules, but this doesn't seem to be the case when an administrator explicity chooses a target hypervisor.

So in the meantime, enter [antiaffinitycheck.py](https://github.com/openstack/osops-tools-contrib/commit/e3b5bc9634c1437ef9538c0a6e7d89c18289b1bb) - a script that OpenStack administrators can use to check whether anti-affinity rules are being adhered to for a given Server Group.  It's pretty straightforward, here's a couple of examples:

```
$ ./antiaffinitycheck.py --list c353197c-5fbb-410f-a7b3-843452a55276
+--------------------------------------+-----------+----------------+
| Instance ID                          | Instance  | Hypervisor     |
+--------------------------------------+-----------+----------------+
| 4c38cf7f-2073-4d96-b377-d2bd29595d8a | app-db-13 | compute2.dev   |
| 7c9d9a8e-1758-4145-a118-b72360eff112 | app-db-10 | compute15.dev  |
| 29d3fe6b-2303-4ce2-af90-636e216ac95e | app-db-4  | compute32.dev  |
| a701afaf-fef5-44f6-9b09-e440ce6b85fe | app-db-14 | compute11.dev  |
| 5c40b1e9-8b38-4978-98b2-391e53e65418 | app-db-11 | compute2.dev   |
| 0b8b19cb-4fae-47c2-b3c7-07b92e11f4a1 | app-db-5  | compute38.dev  |
+--------------------------------------+-----------+----------------+
```

The default output of `nova server-group-list` is a bit awkward and there's no reformatting options available, so `--list` shows us servers and hypervisors involved in a given Server Group in a fairly sane format...  Now if it's not obvious that there's more than one physical hypervisor involved in that group - and there shouldn't be! - we can use `--check`:

```
$ ./antiaffinitycheck.py --check c353197c-5fbb-410f-a7b3-843452a65276

Anti-affinity rules violated in Server Group: c353197c-5fbb-410f-a7b3-843452a55276
+--------------------------------------+-----------+--------------+
| Instance ID                          | Instance  | Hypervisor   |
+--------------------------------------+-----------+--------------+
| 4c38cf7f-2073-4d96-b377-d2bd29595d8a | app-db-13 | compute2.dev |
| 5c40b1e9-8b38-4978-98b2-391e53e65418 | app-db-11 | compute2.dev |
+--------------------------------------+-----------+--------------+
```

If you've any users that make extensive use of these rules then hopefully this script will come in handy for making sure everything's as it should be.
