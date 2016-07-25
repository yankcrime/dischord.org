---
layout: post
title: Cinder multi-backend with multiple Ceph pools
date: 2015-12-22 14:34
published: true
---

This isn't so much of a 'how-to' (as it's been documented with perfect clarity by Sébastien Han [here](http://www.sebastien-han.fr/blog/2013/04/25/ceph-and-cinder-multi-backend/)), it's more of a warning when enabling the multi-backend functionality. _tl;dr_ If you enable Cinder multi-backend, double-check the output of `cinder service-list` and be sure to update the host mappings for all your existing volumes.

I came across a problem recently after enabling Cinder's multi-backend feature to support multiple Ceph pools.  Attempting to delete existing instances that had volumes associated with them would fail; The symptom was a HTTP/504 (gateway timeout) leaving the machine in an 'error' state having sat there 'deleting' for some time.  Instantiating new virtual machines, attaching volumes, and then deleting them was all fine.

For the failed deletions, as far as Nova's logs are concerned the delete mostly goes according to plan apart from the call to Cinder to detach the associated volume:

	DEBUG cinderclient.client [req-3011c2ec-2274-49e4-9f1e-e9ec7eb850a5 ] Failed attempt(1 of
	3), retrying in 1 seconds _cs_request
	/usr/lib/python2.7/dist-packages/cinderclient/client.py:297

And a few seconds later after the third attempt, the following exception makes an appearance:

	ERROR oslo.messaging.rpc.dispatcher [req-3011c2ec-2274-49e4-9f1e-e9ec7eb850a5 ] Exception
	during message handling: Gateway Time-out (HTTP 504)

This is where our instance is left in the error state.  Tracing through from the original request using the request-id (yay for [ELK](https://www.elastic.co/videos/introduction-to-the-elk-stack)) Cinder seemingly gives up and the RPC request times out.  We can see where Nova talks to Cinder API, for example:

	INFO cinder.api.openstack.wsgi [req-c0854007-3b72-4964-b337-b9e331874c4e
	d1a60c7b4bbc427e8f2c3f99d0c2e6d2 509d6ae853114ea9aaaac02804d3a4dd - - -] POST
	http://compute.datacentred.io:8776/v1/509d6ae853114ea9aaaac02804d3a4dd/volumes/7a8a46ba-868a-4620-acc5-84467036fecb/action

There's the message generated:

	DEBUG oslo_messaging._drivers.amqpdriver [req-c0854007-3b72-4964-b337-b9e331874c4e
	d1a60c7b4bbc427e8f2c3f99d0c2e6d2 509d6ae853114ea9aaaac02804d3a4dd - - -] MSG_ID is
	774b48ebff5540fcb9262f35941f6f9d _send
	/usr/lib/python2.7/dist-packages/oslo_messaging/_drivers/amqpdriver.py:311

	DEBUG oslo_messaging._drivers.amqp [req-c0854007-3b72-4964-b337-b9e331874c4e
	d1a60c7b4bbc427e8f2c3f99d0c2e6d2 509d6ae853114ea9aaaac02804d3a4dd - - -] UNIQUE_ID is
	a39a4a8ec681468b87463968a8b64a2c. _add_unique_id
	/usr/lib/python2.7/dist-packages/oslo_messaging/_drivers/amqp.py:252

And then a few seconds later we see another exception:

	ERROR cinder.api.middleware.fault [req-c0854007-3b72-4964-b337-b9e331874c4e
	d1a60c7b4bbc427e8f2c3f99d0c2e6d2 509d6ae853114ea9aaaac02804d3a4dd - - -] Caught error: Timed
	out waiting for a reply to message ID 774b48ebff5540fcb9262f35941f6f9d
	
	[..]

	INFO eventlet.wsgi.server [req-c0854007-3b72-4964-b337-b9e331874c4e d1a60c7b4bbc427e8f2c3f99d0c2e6d2 509d6ae853114ea9aaaac02804d3a4dd - - -] Traceback (most recent call last):
	  File "/usr/lib/python2.7/dist-packages/eventlet/wsgi.py", line 468, in handle_one_response
	    write(b''.join(towrite))
	  File "/usr/lib/python2.7/dist-packages/eventlet/wsgi.py", line 399, in write
	    _writelines(towrite)
	  File "/usr/lib/python2.7/socket.py", line 334, in writelines
	    self.flush()
	  File "/usr/lib/python2.7/socket.py", line 303, in flush
	    self._sock.sendall(view[write_offset:write_offset+buffer_size])
	  File "/usr/lib/python2.7/dist-packages/eventlet/greenio.py", line 376, in sendall
	    tail = self.send(data, flags)
	  File "/usr/lib/python2.7/dist-packages/eventlet/greenio.py", line 358, in send
	    total_sent += fd.send(data[total_sent:], flags)
	error: [Errno 104] Connection reset by peer

There were no other problems on the platform at the time - loadbalancers are fine, RabbitMQ is OK - and so on the face of it this is something of a mystery.

My initial attempts at triaging the issue involved manually trying to detach the volume associated with the instance:

	❯ nova reset-state --active 75966ed2-e003-4575-8b64-08c4c414492c
	Reset state for server 75966ed2-e003-4575-8b64-08c4c414492c succeeded; new state is active

	❯ nova stop 75966ed2-e003-4575-8b64-08c4c414492c
	Request to stop server 75966ed2-e003-4575-8b64-08c4c414492c has been accepted.

	❯ nova volume-detach 75966ed2-e003-4575-8b64-08c4c414492c
	7a8a46ba-868a-4620-acc5-84467036fecb

	❯ cinder show 7a8a46ba-868a-4620-acc5-84467036fecb | grep -i status
	| status | detaching |

And there it hung, indefinitely.  Googling the symptoms of the problem don't necessarily get you very far at this point, as there's [no](https://bugs.launchpad.net/nova/+bug/1449221) [shortage](https://bugs.launchpad.net/cinder/+bug/1413610) of [bugs](https://ask.openstack.org/en/question/62198/how-to-debug-volume-stuck-in-deleting-state-issue/) out there to do with Cinder volumes ending up in a strange state and instances that can't be deleted.  Most of these suggest [diving into the database](https://ask.openstack.org/en/question/66918/how-to-delete-volume-with-available-status-and-attached-to/) in order to clean things up but really you should try and discern the root cause if it becomes apparent that this isn't a one off.

The clue really is in the fact that there's some kind of messaging timeout.  We know that `cinder-api` is receiving the request from `nova-compute`, so what's happening to the message?  Why isn't `cinder-volume` picking it up?  Well, every service on OpenStack has a registry of what runs where in terms of agents, whether it's `nova-conductor`, `nova-compute`, `neutron-dhcp-agent`, `neutron-metadata-agent`, etc., and the same is true of Cinder - there's `cinder-volume` and `cinder-scheduler`. Knowing that there'd been some very recent changes to Cinder's configuration, on a hunch I decided to check the output of `cinder service-list`:

	❯ cinder service-list
	+------------------+-----------------------------------------------+----------+-------+
	|      Binary      |                      Host                     |  Status  | State |
	+------------------+-----------------------------------------------+----------+-------+
	| cinder-scheduler |                   controller0                 | enabled  |   up  |
	| cinder-scheduler |                   controller1                 | enabled  |   up  |
	|  cinder-volume   |                   controller0                 | enabled  |  down |
	|  cinder-volume   |                   controller1                 | enabled  |  down |
	|  cinder-volume   | rbd:cinder.volumes.flash@cinder.volumes.flash | enabled  |   up  |
	|  cinder-volume   |       rbd:cinder.volumes@cinder.volumes       | enabled  |   up  |
	+------------------+-----------------------------------------------+----------+-------+

And there is our smoking gun.  Enabling the multi-backend functionality has introduced two new instances of `cinder-volume`, one per pool, and now the previous instances with the old configuration that were responsible for talking to Ceph are now 'down' - that's why the RPC messages from `cinder-api` aren't being responded to, because the 'host' responsible for that volume is AWOL.

In order to fix this there's a couple of things that need to be done.  First is to administratively disable the older, now redundant, services:

	❯ for host in controller{0,1}; do cinder service-disable $host cinder-volume ; done
	+-------------+---------------+----------+
	|    Host     |     Binary    |  Status  |
	+-------------+---------------+----------+
	| controller0 | cinder-volume | disabled |
	+-------------+---------------+----------+
	+-------------+---------------+----------+
	|    Host     |     Binary    |  Status  |
	+-------------+---------------+----------+
	| controller1 | cinder-volume | disabled |
	+-------------+---------------+----------+

The next step is to update the mappings for the volumes themselves, and unfortunately to do that we need to make some changes in the Cinder database.  The first change is to the `volumes` table, and the second is to the `volume_mappings` table - both of these will update the associated `host` column for the row that corresponds to the `volume_id` associated with this instance.

	MariaDB [cinder]> select host from volumes where id='7a8a46ba-868a-4620-acc5-84467036fecb';
	+---------------------+
	| host                |
	+---------------------+
	| controller1#DEFAULT |
	+---------------------+

	MariaDB [cinder]> select volume_id,attached_host,instance_uuid from volume_attachment where
	volume_id='7a8a46ba-868a-4620-acc5-84467036fecb';
	+--------------------------------------+---------------------+--------------------------------------+
	| volume_id                            | attached_host       | instance_uuid			        	|
	+--------------------------------------+---------------------+--------------------------------------+
	| 7a8a46ba-868a-4620-acc5-84467036fecb | controller1#DEFAULT | 75966ed2-e003-4575-8b64-08c4c414492c |
	+--------------------------------------+---------------------+--------------------------------------+

The convention when updating the `host` and `attached_host` columns is to use `host#pool`, so to fix these rows we need to do the following:

	MariaDB [cinder]> update volumes set host='rbd:cinder.volumes@cinder.volumes#cinder.volumes'
	where id='7a8a46ba-868a-4620-acc5-84467036fecb' limit 1;
	MariaDB [cinder]> update volume_attachment set
	attached_host='rbd:cinder.volumes@cinder.volumes#cinder.volumes' where
	volume_id='7a8a46ba-868a-4620-acc5-84467036fecb' limit 1;

Now let's try deleting our instances again:

	❯ cinder reset-state --state in-use 7a8a46ba-868a-4620-acc5-84467036fecb
	❯ nova delete 75966ed2-e003-4575-8b64-08c4c414492c
	Request to delete server 75966ed2-e003-4575-8b64-08c4c414492c has been accepted.

And a couple of seconds later:

	❯ nova show 75966ed2-e003-4575-8b64-08c4c414492c
	ERROR (CommandError): No server with a name or ID of '75966ed2-e003-4575-8b64-08c4c414492c'
	exists.

Hurrah!  If that works you'll need to update both the `volumes` and `volume_attachments` table en masse to reflect the new hosts, and then delete the service entries for the redundant services themselves.
