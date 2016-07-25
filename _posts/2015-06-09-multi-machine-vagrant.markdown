---
layout: post
title: Multi-machine Vagrant
date: 2015-06-09 15:23
published: false
categories:
---

I've seen a few different ways of handling the provisioning of multiple machines in [Vagrant](http://vagrantup.com), such as [Scott Lowe's](http://blog.scottlowe.org)'s approach of either specifying them in-line in the
Vagrantfile or including a YAML file and then parsing values accordingly.

The best option I've seen, and the approach we've used here at [DataCentred](http://www.datacentred.co.uk), is to make use of a Vagrant plugin called [Nugrant](https://github.com/maoueh/nugrant).  This plugin abstracts away various settings that you can conditionally refer to in your Vagrantfile, and then have a per-user configuration 
