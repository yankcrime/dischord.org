---
layout: post
title: Cleaning up after Neutron
date: 2016-01-05 17:42
published: true
categories: OpenStack
---

_I've cobbled this post together from some older notes - there's a few things in here that haven't
been a problem (for [us](http://www.datacentred.co.uk)) since Juno, and with recent changes to
Neutron i.e DVR and L3-HA some of it is lapsing into irrelevance anyway. But seeing as it came up
recently on [IRC](https://wiki.openstack.org/wiki/IRC) I thought I'd put it together here just in
case anyone else needs some help._

Managing orphaned Neutron objects is something that most operators are probably going to have to do
at some point.  Whether it's related to a configuration issue or you've had some kind of problem
with Neutron itself, you can often end up with network namespaces on your network nodes that are
effectively redundant, hogging resources that could otherwise be used elsewhere.

As an example, [the default configuration in Juno](https://bugs.launchpad.net/neutron/+bug/1052535)
when installing from Ubuntu's packages was to not delete namespaces after their associated network
or router was removed.  If you didn't keep tabs on this, you'd soon end up with a lot of redundant
namespaces on your network nodes.  As a public cloud operator this is especially problematic when
you've got public IPv4 address space to manage and you really don't want precious addresses being
wasted on gateway interfaces for virtual routers that are no longer in use.

So what to do?

* [Identifying orphans](#identifying)
* [Cleaning up routers](#routers)
* [Cleaning up networks](#networks)
* [Cleaning up namespaces](#namespaces)

## <a name="identifying"></a>Identifying orphans

I wrote a [small Python script](http://dischord.org/2015/04/14/openstack-orphans) a while back that
helps to do exactly this.  It's [now a part of the official OSOps
repo](https://github.com/openstack/osops-tools-generic/blob/master/neutron/listorphans.py), and
using it we can do something like:

```
❯ ./listorphans.py routers
45 orphan(s) found of type routers [03fe3926-167c-4460-a139-12335615a02c,
096bfb34-2df0-4781-993d-5a0edb0db179, 16a48cf4-940d-4221-9427-e8037a223bb4
<snip>
```

If you wanted to double check some of these before you go ahead and delete anything, that's both
sensible and easy.  The script is calling the Neutron and Keystone APIs, and for each object
(routers in our case), it's seeing if the associated tenant ID is valid.  Let's do the same from the
CLI:

```
❯ neutron router-show 096bfb34-2df0-4781-993d-5a0edb0db179
+-----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                 | Value                                                                                                                                                                                      |
+-----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                                                                                                                       |
| distributed           | False                                                                                                                                                                                      |
| external_gateway_info | {"network_id": "6751cb30-0aef-4d7e-94c3-ee2a09e705eb", "enable_snat": true, "external_fixed_ips": [{"subnet_id": "2af591ca-48ac-42b7-afc6-e691b3aa4c8a", "ip_address": "185.98.148.134"}]} |
| ha                    | False                                                                                                                                                                                      |
| id                    | 096bfb34-2df0-4781-993d-5a0edb0db179                                                                                                                                                       |
| name                  | default                                                                                                                                                                                    |
| routes                |                                                                                                                                                                                            |
| status                | ACTIVE                                                                                                                                                                                     |
| tenant_id             | 6ca0dd9ace034080853781c411f8e7a8                                                                                                                                                           |
+-----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

```
❯ openstack project show 6ca0dd9ace034080853781c411f8e7a8
No tenant with a name or ID of '6ca0dd9ace034080853781c411f8e7a8' exists.
```

Finding redundant Neutron networks is just a case of running the script with the `networks` option
instead:

```
❯ ./listorphans.py networks
1 orphan(s) found of type networks
5fe2e974-10b4-4f5f-a678-303943137497
```

Again, if you want to verify this then you just need to do `neutron net-show
5fe2e974-10b4-4f5f-a678-303943137497` followed by `openstack project show` with the UUID of the
associated tenant.

Now that we've identified some orphans, we need to clean them up.

## Cleaning up
### <a name="routers"></a>Routers

Deleting an orphaned router can be a little involved depending on how many interfaces it has
defined.  In this example we'll delete a router that has a gateway interface (i.e an Internet-facing
IP address) and a port in a private subnet.  This is probably the most common configuration.

If we look at an example router's configuration:

```
❯ neutron router-show 16a48cf4-940d-4221-9427-e8037a223bb4
+-----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                 | Value                                                                                                                                                                                      |
+-----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                                                                                                                       |
| distributed           | False                                                                                                                                                                                      |
| external_gateway_info | {"network_id": "6751cb30-0aef-4d7e-94c3-ee2a09e705eb", "enable_snat": true, "external_fixed_ips": [{"subnet_id": "2af591ca-48ac-42b7-afc6-e691b3aa4c8a", "ip_address": "185.98.149.130"}]} |
| ha                    | False                                                                                                                                                                                      |
| id                    | 16a48cf4-940d-4221-9427-e8037a223bb4                                                                                                                                                       |
| name                  | default                                                                                                                                                                                    |
| routes                |                                                                                                                                                                                            |
| status                | ACTIVE                                                                                                                                                                                     |
| tenant_id             | 06348371a09148d194354183108708f8                                                                                                                                                           |
+-----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

The first thing to do is remove the gateway interface.  This is straightforward:

```
❯ neutron router-gateway-clear 16a48cf4-940d-4221-9427-e8037a223bb4
Removed gateway from router 16a48cf4-940d-4221-9427-e8037a223bb4
❯ neutron router-show 16a48cf4-940d-4221-9427-e8037a223bb4
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| distributed           | False                                |
| external_gateway_info |                                      |
| ha                    | False                                |
| id                    | 16a48cf4-940d-4221-9427-e8037a223bb4 |
| name                  | default                              |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | 06348371a09148d194354183108708f8     |
+-----------------------+--------------------------------------+
```

Note that the second time we run the `neutron router-show` command there's now no information
displayed for the `external_gateway_info`.  Now we need to delete the internally-facing ports, so
let's find out exactly what remaining ports are enabled on this router:

```
❯ neutron router-port-list 16a48cf4-940d-4221-9427-e8037a223bb4
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------------+
| id                                   | name | mac_address       | fixed_ips                                                                          |
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------------+
| ff468adc-1bf0-4bf5-b655-d757fd258047 |      | fa:16:3e:1a:fe:fb | {"subnet_id": "758f9ba5-cbb7-4667-9783-ee3c9b5bee98", "ip_address": "192.168.0.1"} |
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------------+
```

With that information, specifically the associated subnet ID, we can now delete the port:

```
❯ neutron router-interface-delete 16a48cf4-940d-4221-9427-e8037a223bb4 758f9ba5-cbb7-4667-9783-ee3c9b5bee98
Removed interface from router 16a48cf4-940d-4221-9427-e8037a223bb4.
```

```
❯ neutron router-delete 16a48cf4-940d-4221-9427-e8037a223bb4
Deleted router: 16a48cf4-940d-4221-9427-e8037a223bb4
```

I'm lazy, so here's a script to do the whole thing for us:

```
#!/usr/bin/env bash
ROUTER=$1
neutron router-gateway-clear $ROUTER
for PORT in $(neutron router-port-list -F fixed_ips $ROUTER | awk '{ print $3 }' | tr -d '\n|",') ; do
        neutron router-interface-delete $ROUTER $PORT
done
neutron router-delete $ROUTER
```

Now it's just a case of wrapping that up in a quick for loop for each of the orphaned router UUIDs
we collected earlier.

### <a name="networks"></a>Networks

Once you've got the routers out of the way, it's mostly a simple matter of deleting the networks
with `neutron net-delete $NET_UUID`.  If you see the following error:

```
❯ ./neutronlistorphans.py networks
1 orphan(s) found of type networks
5fe2e974-10b4-4f5f-a678-303943137497
❯ neutron net-delete 5fe2e974-10b4-4f5f-a678-303943137497
Unable to complete operation on network 5fe2e974-10b4-4f5f-a678-303943137497. There are one or more ports still in use on the network.
```

Then, as it suggests, something still has an interface plumbed into that network.  If it's not a
router then it could be a VM, but you can find out what exactly by doing the following:

```
❯ neutron net-show -F subnets 5fe2e974-10b4-4f5f-a678-303943137497
+---------+--------------------------------------+
| Field   | Value                                |
+---------+--------------------------------------+
| subnets | 96554e99-9eca-40ac-af1c-1ff1870fed0f |
+---------+--------------------------------------+
❯ neutron port-list --all-tenants -c id -c fixed_ips | grep 96554e99-9eca-40ac-af1c-1ff1870fed0f
| 5badcd73-a7cb-4e65-a368-254458b1b203 | {"subnet_id": "96554e99-9eca-40ac-af1c-1ff1870fed0f", "ip_address": "192.168.0.1"}     |
| 6938fff5-9b6c-4bb5-bf81-d47b6fbc44f9 | {"subnet_id": "96554e99-9eca-40ac-af1c-1ff1870fed0f", "ip_address": "192.168.0.3"}     |
```

And then doing `neutron port-show $UUID` on each of those ports will tell you what's still lingering.  In
my case it's a router and a virtual machine.  Oops - I should probably get rid of those as well.

## <a name="namespaces"></a>Namespaces

What about orphaned namespaces then?  Here's a quick script that'll identify any L3
(`qrouter-`) namespaces across a number of network nodes that Neutron doesn't know about
(and so is no longer managing):

```
#!/usr/bin/env bash
for netnode in $1 ; do
	echo $netnode
	for router in $(ssh $netnode 'ip netns list | grep qrouter | cut -d - -f 2-20') ; do
		neutron router-show $router | grep -i unable
	done
done
```

Just call it with a list of network nodes and it should return the invalid namespaces.

For L2 namespaces (`qdhcp-`), use this slightly altered version instead:

```
#!/usr/bin/env bash
for netnode in $1 ; do
	echo $netnode
	for router in $(ssh $netnode 'ip netns list | grep qdhcp | cut -d - -f 2-20') ; do
		neutron net-show $router | grep -i unable
	done
done
```

Now, for each of the invalid `qdhcp` or `qrouter` namespaces we need to manually delete the network
namespace itself as well as the corresponding port(s) in OVS. In this example I'm on a network node
called `osnet1` and deleting a router whose UUID is `1e944eb4-2773-40f1-9a03-7403f915b334`:

```
nick@osnet1:~$ sudo ip netns exec qrouter-1e944eb4-2773-40f1-9a03-7403f915b334 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2956: qg-51fde206-37: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether fa:16:3e:df:ea:ee brd ff:ff:ff:ff:ff:ff
    inet 185.98.149.219/23 brd 185.98.149.255 scope global qg-51fde206-37
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fedf:eaee/64 scope link
       valid_lft forever preferred_lft forever
```

Here we have a router with a single (gateway) interface configured - `qg-51fde206-37`. Verify that there's an OVS port we need to delete as well:

```
nick@osnet1:~$ sudo ovs-vsctl show | grep -A5 -B5 qg-51fde206-37
        type: internal
Port "qg-eebb6dbe-9a"
    tag: 1
    Interface "qg-eebb6dbe-9a"
        type: internal
Port "qg-51fde206-37"
    tag: 1
    Interface "qg-51fde206-37"
        type: internal
Port "qr-e724f3da-db"
    tag: 113
    Interface "qr-e724f3da-db"
        type: internal
```

The fact that it's tagged with '1' is another sign that it's invalid. In the
case of a DHCP namespace it'll have a 'tap' interface and if it's invalid
it'll have a tag of 4095. Now we can go ahead and delete the namespace and then
the OVS port with confidence:


```
nick@osnet1:~$ sudo ip netns delete qrouter-1e944eb4-2773-40f1-9a03-7403f915b334
nick@osnet1:~$ sudo ovs-vsctl del-port qg-51fde206-37
nick@osnet1:~$ sudo ip netns exec qrouter-1e944eb4-2773-40f1-9a03-7403f915b334 ip a
Cannot open network namespace "qrouter-1e944eb4-2773-40f1-9a03-7403f915b334": No such file or directory
nick@osnet1:~$ sudo ovs-vsctl show | grep qg-51fde206-37
nick@osnet1:~$
```

Done.  Oh too many steps you say?  Here's another script that does the work for you (for routers
anyway):

```
#!/usr/bin/env bash

ROUTER=$1
GIF=$(sudo ip netns exec qrouter-$ROUTER ip a | grep qg | awk '{ print $2 }' RS="\n\n" | cut -d : -f 1)
RIF=$(sudo ip netns exec qrouter-$ROUTER ip a | grep qr | awk '{ print $2 }' RS="\n\n" | cut -d : -f 1)

echo "Deleting network namespace qrouter-$1"
ip netns delete qrouter-$ROUTER

echo "Deleting OVS ports $GIF $RIF"
if [[ $GIF ]]; then ovs-vsctl del-port $GIF ; fi
if [[ $RIF ]]; then ovs-vsctl del-port $RIF ; fi
```

In fact it's because these things have to be done in a specific order that this is sometimes the
reason why you might still have some namespaces hanging around, even if you've configured Neutron's
L3 agent to delete them automatically.  It's not impossible for additional interfaces (that Neutron
doesn't know about) to be defined in a namespace, which will then cause the auto-delete functionality
to fail.

Deleting the `qdhcp` namespaces follows a similar process, although they're normally a little less
problematic as you shouldn't have any additional L3 interfaces configured. Simply do the `ip netns
delete` step for each offending namespace, along with any associated OVS ports.

