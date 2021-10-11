---
layout: page
permalink: /openstack16-101-workshop/exercise01
---
## Exercise 1 - Looking around

By now you should be logged into the *Bastion host* in a terminal.  Not much runs on the bastion host, so we'll first *ssh* somewhere more interesting.

```[student@workstation-f3db ~]$ ssh stack@undercloud.example.com```

You should now be logged into the undercloud workstation.

```
Activate the web console with: systemctl enable --now cockpit.socket

This system is not registered to Red Hat Insights. See https://cloud.redhat.com/
To register this system, run: insights-client --register

Last login: Sun Oct 10 06:04:03 2021 from 192.0.2.252
[stack@undercloud ~]$
```

The *undercloud* is a small instance of OpenStack that runs the *Director*.  The *undercloud* contains the necessary components of OpenStack to deploy and manage an *overcloud*, which is where the workloads run.

We will not cover the *undercloud* in this workshop.  This workshop is designed to give you a small taste of running a workload in **OpenStack**, and therefore will be dealing exclusively with the *overcloud*.  So let's switch to that.

```
[stack@undercloud ~]$ source ~/overcloudrc
(overcloud) [stack@undercloud ~]$
```

Let's look around the overcloud for a minute.

*Projects* in OpenStack are organisational units in the cloud to which you can assign users.  A project is created for each set of instances and networks that are configured as a discrete entity for the project.  Projects are therefore restricted views into OpenStack resources.  We have set up a project for each user in the lab:

```
(overcloud) [stack@undercloud ~]$ openstack project list
+----------------------------------+---------------+
| ID                               | Name          |
+----------------------------------+---------------+
| 125a825e987c40309a1ce4f8f16a0859 | admin         |
| 239f015f3d264e3595b9fe18f9120110 | user7project  |
| 3efbd4c99a6e4a49a9a32ce371bfeefd | user8project  |
| 4eb7f64b9d0d4429952b945648044bac | user3project  |
| 62495357e0b243c3a46d6e4f5af150f5 | user4project  |
| 8a108c059bd940d78c6ac3c2cda22e16 | user5project  |
| ca51b4bb1b554f0c892ce31327505af1 | user9project  |
| d0130e333e11477a9cb331ffbd6cf218 | user10project |
| d47243d6438e4a969554d916813323d0 | user6project  |
| d88c6df92a0a4020915c595032bcd379 | user1project  |
| da48abd02a3d41ed94717457795d9512 | service       |
| f644255cdea9482ba50da6d3c8b0c5f3 | user2project  |
+----------------------------------+---------------+
```

We'll switch to your project in a second.  But first, let's continue to explore.  The lab is also set up with a public network:

```
(overcloud) [stack@undercloud ~]$ openstack network list
+--------------------------------------+-------------+--------------------------------------+
| ID                                   | Name        | Subnets                              |
+--------------------------------------+-------------+--------------------------------------+
| 019f687e-667e-4db2-9403-aa2a5b12dd51 | public      | b3af55a2-6f0a-4b23-b5f8-ff111adbbc76 |
| 0202c63a-c446-414b-a1cc-2b11158bda9f | lb-mgmt-net | fdecae96-6d3c-492e-87fa-6531c3c845a1 |
+--------------------------------------+-------------+--------------------------------------+
```

The public network is attached to the underlying phsyical provider network, and therefore is the bridge outside of the cluster.

There is a public subnet (attached to the *public* network) created in the range *10.0.0.0/24*:

```
(overcloud) [stack@undercloud ~]$ openstack subnet list
+--------------------------------------+----------------+--------------------------------------+---------------+
| ID                                   | Name           | Network                              | Subnet        |
+--------------------------------------+----------------+--------------------------------------+---------------+
| b3af55a2-6f0a-4b23-b5f8-ff111adbbc76 | public-subnet  | 019f687e-667e-4db2-9403-aa2a5b12dd51 | 10.0.0.0/24   |
| fdecae96-6d3c-492e-87fa-6531c3c845a1 | lb-mgmt-subnet | 0202c63a-c446-414b-a1cc-2b11158bda9f | 172.24.0.0/16 |
+--------------------------------------+----------------+--------------------------------------+---------------+
```

All public-facing IP address (ie, address that you can ping) created in this lab will be created in this 10.0.0.X network.

You can list the current instances:

```
(overcloud) [stack@undercloud ~]$ openstack server list

(overcloud) [stack@undercloud ~]$
```

Notice that this list is currently empty because there are no instances at this point.

We can list the compute nodes that your instances will land on when we create them:

```
(overcloud) [stack@undercloud ~]$ openstack hypervisor list
+--------------------------------------+-------------------------------------+-----------------+--------------+-------+
| ID                                   | Hypervisor Hostname                 | Hypervisor Type | Host IP      | State |
+--------------------------------------+-------------------------------------+-----------------+--------------+-------+
| f73c35e2-2152-47b1-b387-1801dd4aa4f7 | overcloud-novacompute-1.localdomain | QEMU            | 172.17.0.212 | up    |
| 422da7d9-de7f-4ca4-92ea-40a1c20fb0cf | overcloud-novacompute-0.localdomain | QEMU            | 172.17.0.211 | up    |
+--------------------------------------+-------------------------------------+-----------------+--------------+-------+
```

When you create an *instance*, the resources allocated to the instance is determined by the *flavor* used to create it.  The lab already has a flavor set up for you:

```
(overcloud) [stack@undercloud ~]$ openstack flavor list
+--------------------------------------+---------+-----+------+-----------+-------+-----------+
| ID                                   | Name    | RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+---------+-----+------+-----------+-------+-----------+
| 0ba4fe89-115b-4af5-a6ea-52fccff31f62 | m1.nano | 128 |    1 |         0 |     1 | True      |
+--------------------------------------+---------+-----+------+-----------+-------+-----------+
```

You'll notice that this flavor was created in the *admin* project and set up as *Public*.  Therefore it is visible from all projects.

[Previous Exercise](setup) / [Next Exercise](exercise02)
