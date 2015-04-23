---
layout: post
title: Orchestrating CoreOS with OpenStack Heat
date: 2015-04-18 17:53
published: true
categories:
---

Having finally spent a bit of time with [OpenStack's Heat](https://wiki.openstack.org/wiki/Heat), I've started to see what I can do with automating infrastructure deployments and services by using it in conjunction with [CoreOS](http://coreos.org). This post sort of builds on [Scott Lowe's introduction to CoreOS and Heat](http://blog.scottlowe.org/2014/08/13/deploying-coreos-on-openstack-using-heat/) and does a few of the things he suggests, such as creating a dedicated network and deploying an arbitrary number of instances.  It's just enough to get a cluster stood up with which you can then define some services and roll out your application stack in order to start testing.

The Heat template itself looks like this:

{% gist af2a9dce692014dd38b8 %}

Most of it is pretty standard, the interesting bits that I think are worth pointing out are:

* Line 22, or thereabouts, where I list a few flavors.  These are specific to [DataCentred's OpenStack installation](http://www.datacentred.co.uk/on-demand-compute/) and will need changing if you're deploying this elsewhere;
* The image ID on line 35 is also specific to [DataCentred](http://www.datacentred.co.uk) but can be overridden either here or as a parameter when you create the stack;
* Line 99, where we define a ResourceGroup with a count passed in as a parameter;
* Line 111, which saves confusion by suffixing the (parameterised) name with the RG's index value for the OS::Nova::Server instance;
* The `user_data` section which does just about enough to start [etcd](https://coreos.com/etcd/) and [fleet](https://github.com/coreos/fleet) and gets our instances talking to one another.


To launch this stack using the `heat` CLI, run the following command:

```
$ heat stack-create -f coreos-heat.yaml \
-P key_name=deadline \
-P count=3 \
-P public_net_id=6751cb30-0aef-4d7e-94c3-ee2a09e705eb \
-P discovery_url=$(curl -sw "\n" 'https://discovery.etcd.io/new?size=3') \
-P name=webserver coreos
```

In that example, `webserver` is the prefix that each instance will use and the last argument, `coreos`, is the name of the stack itself.  And yes, passing in `count=3` is a bit redundant as it's the default in the template, but for illustration's sake I think it helps here ;) The `discovery_url` is passed in as a parameter, and in our example and in my lab I've been using the etcd project's provided discovery service, but you're free to run your own instead of course.

Kick that command off, give it a few minutes, and eventually you should have a successfully deployed stack.  Login to one of the instances and then you'll be able to verify the state of the cluster:

```
nick@deadline:~> ssh core@185.43.218.192
Last login: Sat Apr 18 18:06:22 2015 from 86.143.53.8
CoreOS stable (607.0.0)
core@webserver-0 ~ $ fleetctl list-machines
MACHINE		IP		METADATA
1fc4fbb3...	192.168.10.24	-
59c16e8a...	192.168.10.23	-
882768f6...	192.168.10.22	-
```

At this point you can define some units and launch containers in your cluster via `fleet` - the CoreOS project's website has you covered with a good introduction [to get you started](https://coreos.com/docs/launching-containers/launching/launching-containers-fleet/).
