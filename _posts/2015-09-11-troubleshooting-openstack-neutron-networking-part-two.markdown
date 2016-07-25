---
layout: post
title: Troubleshooting OpenStack Neutron Networking, Part Two
date: 2015-09-11 19:24
published: false
categories:
---

It's taken me a little longer than I originally anticipated to write up the second post on this
series in OpenStack Neutron network troubleshooting, mainly because of the fact that with an
impending upgrade to Kilo around the corner and the changes to L3 architecture that DVR and L3-HA
bring about, I didn't want to spend time drafting something that'd soon be out of date.  Such is the
nature of IT however that basically anything I write technology related will be out of date in no
time anyway... ;)

I figured I'd do this part two then talking about tips and tricks for L3 troubleshooting with a
'vanilla' Neutron / OVS installation without any DVR or L3-HA in place and then in part three I can
talk about the upgrade approach and anything else that stands out and needs special attention.

## Typical problem scenarios
User issues with networking in OpenStack generally get reported as something along the lines of:

1. "I've booted an instance but it hasn't got an IP address";
2. "I've booted an instance but I can't ping anything";
3. "I've booted an instance, assigned a floating IP, but can't ping or connect to it".

For the first issue you should refer to my DHCP troubleshooting notes [here](http://dischord.org/2015/03/09/troubleshooting-openstack-neutron-networking-part-one/), but aside from 
an occassionaly borked Security Group rule the other two can send you off on quite the ride before
you track down the root cause.

## Information gathering

As with part one of this guide, there's a few key bits of information you should arm yourself with
before you attempt to jump into the rabbit hole that is Neutron network troubleshooting.  From an L3
standpoint, it's useful to know:

* Project (tenant) UUID;
* Associated router UUIDs;
* Floating IP UUIDs.

With those three bits of information to hand we can start to do some digging.

## Architecture



