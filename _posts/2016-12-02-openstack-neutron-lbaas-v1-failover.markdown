---
layout: post
title: OpenStack Neutron LBaaS v1 Failover
date: 2016-12-02 15:49
published: true
categories:
---

A quick note on what to do if you find yourself in the unfortunate position of having to failover an [LBaaS](https://wiki.openstack.org/wiki/Neutron/LBaaS) V1 pool from one agent to another.  This isn't supported via the API so you need to roll up your sleeves and dust off your favourite MySQL client.

In this example, `nn2` has failed and everything else has gracefully migrated away - I'm just stuck with some LBaaS pools that I need to move.  In case you didn't realise, you can figure out your Neutron agent UUID by doing the following:

```
$ openstack network agent list | grep -iE 'loadbalancer.*nn2'
| 9dfa45dd-4562-4180-b7da-879b8c539e5f | Loadbalancer agent | nn2 | None | False | UP | neutron-lbaas-agent |
```

In this case, the 'False' column means it's dead and so we need to move all pools associated with that agent.  You can see which pools are affected by doing:

```
$ neutron lb-pool-list-on-agent 44770887-a458-4503-9648-d07f97f56d96
+--------------------------------------+---------+-------------------+----------+----------------+--------+
| id                                   | name    | lb_method         | protocol | admin_state_up | status |
+--------------------------------------+---------+----------+--------+----------+----------------+--------+
| 00662f74-3fd6-5162-af12-8ee15b73232e | lb1     | LEAST_CONNECTIONS | TCP      | True           | ACTIVE |
| 0976e919-5305-2232-b924-97176a1abbcb | lb2     | ROUND_ROBIN       | TCP      | True           | ACTIVE |
| 0a628525-3708-abc2-ac87-60c0ef4d66f5 | lb3     | ROUND_ROBIN       | TCP      | True           | ACTIVE |
| 0b57422f-052f-d124-ac31-49b962cd825e | lb4     | ROUND_ROBIN       | TCP      | True           | ACTIVE |
| 10095dbe-1568-bb2c-923e-d9f01241a838 | lb5     | ROUND_ROBIN       | TCP      | True           | ACTIVE |

[ .. ]
```

You'll then want to select a suitable target for where to migrate these pools to.  In my example, it's `nn5`:

```
$ openstack network agent list | grep -iE 'loadbalancer.*nn5'
| ee998c47-6a6c-4632-b4d5-64612523823b | Loadbalancer agent | nn5 | None | True  | UP | neutron-lbaas-agent |
```

The table in question which manages the pool to agent mapping is `poolloadbalanceragentbindings`:

```
MariaDB [neutron]> describe poolloadbalanceragentbindings;
+----------+-------------+------+-----+---------+-------+
| Field    | Type        | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| pool_id  | varchar(36) | NO   | PRI | NULL    |       |
| agent_id | varchar(36) | NO   | MUL | NULL    |       |
+----------+-------------+------+-----+---------+-------+
2 rows in set (0.00 sec)
```

> Make sure you take a backup before doing any updates!

We need to update the rows in this table so that the target agent UUID is responsible for the pools that need migrating.

Let's verify how many we have to deal with:

```
MariaDB [neutron]> select * from poolloadbalanceragentbindings where agent_id = '9dfa45dd-4562-4180-b7da-879b8c539e5f';

[..]

35 rows in set (0.00 sec)
```

Now let's update this database table and change that table ID en masse:

```
MariaDB [neutron]> update poolloadbalanceragentbindings set agent_id = 'ee998c47-6a6c-4632-b4d5-64612523823b' where agent_id = '9dfa45dd-4562-4180-b7da-879b8c539e5f' limit 35;
```

> Protip:  Always qualify with the `limit` statement.  This can often stop or limit the damage from a botched query!

With that done, we need kick Neutron in order for the new agent to realise that there's a whole load of pools it should be responsible for.  You need to restart `neutron-server` first of all, and then the target `neutron-lbaas-agent` in question.  

Keep an eye on your logs and at this point you should see your pools come back to life on your target network node.
