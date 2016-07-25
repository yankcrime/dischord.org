---
layout: post
title: OpenStack Orphans
date: 2015-04-14 10:39
published: true
categories:
---

It's surprisingly easy to find yourself with a large number of orphaned Neutron objects in your OpenStack installation, i.e resources left behind after a project or tenant has been deleted.  This can be particularly problematic when it comes to routers and floating IP addresses as these can quickly eat into what precious few addresses you might have available, especially if they're publically accessible.

Here's a bit of python I've knocked together to check for orphaned resources of this type within your OpenStack install.  The credentials are taken from your current working environment and you'll need to be an administrator in order for this to work.

{% gist 16e9fec07bd1210209cf %}

