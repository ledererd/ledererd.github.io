---
layout: page
permalink: /openstack16-101-workshop/exercise02
---
# Exercise 2 - Create a basic network

Since we're segregating students in the lab by *projects*, let's now switch to the the project allocated to you.

The details of the project are encoded in a configuration file.  The configuration files are located at */home/stack/userX/userX
rc*.  You must "dot" or "source" this file to switch to the project.  For example, if you are assigned **user1**, run the following:

```
(overcloud) [stack@undercloud ~]$ source /home/stack/user1/user1rc
(user1) [stack@undercloud ~]$
```

You will notice in this restricted view, you can only see the artifacts allocated to this project:

```
(user1) [stack@undercloud ~]$ openstack project list
+----------------------------------+--------------+
| ID                               | Name         |
+----------------------------------+--------------+
| d88c6df92a0a4020915c595032bcd379 | user1project |
+----------------------------------+--------------+

(user1) [stack@undercloud ~]$ openstack hypervisor list
Policy doesn't allow os_compute_api:os-hypervisors to be performed. (HTTP 403) (Request-ID: req-9ec7f172-7e6a-4384-bd02-b7fdcd581c44)

(user1) [stack@undercloud ~]$ openstack network list
+--------------------------------------+--------+--------------------------------------+
| ID                                   | Name   | Subnets                              |
+--------------------------------------+--------+--------------------------------------+
| 019f687e-667e-4db2-9403-aa2a5b12dd51 | public | b3af55a2-6f0a-4b23-b5f8-ff111adbbc76 |
+--------------------------------------+--------+--------------------------------------+
```

You'll notice above that the public network is visible in all projects.

When we create instances, we'll create them on a private network, because we want to control when we expose them to the outside world.  Let's create the private network now:

```
(user1) [stack@undercloud ~]$ openstack network create private
+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                     | Value                                                                                                                                                                   |
+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up            | UP                                                                                                                                                                      |
| availability_zone_hints   |                                                                                                                                                                         |
| availability_zones        |                                                                                                                                                                         |
| created_at                | 2021-10-10T10:57:03Z                                                                                                                                                    |
| description               |                                                                                                                                                                         |
| dns_domain                |                                                                                                                                                                         |
| id                        | 6c1c797c-0dfa-449c-b20f-974a0b8e71bc                                                                                                                                    |
| ipv4_address_scope        | None                                                                                                                                                                    |
| ipv6_address_scope        | None                                                                                                                                                                    |
| is_default                | False                                                                                                                                                                   |
| is_vlan_transparent       | None                                                                                                                                                                    |
| location                  | cloud='', project.domain_id=, project.domain_name='Default', project.id='d88c6df92a0a4020915c595032bcd379', project.name='user1project', region_name='regionOne', zone= |
| mtu                       | 1442                                                                                                                                                                    |
| name                      | private                                                                                                                                                                 |
| port_security_enabled     | True                                                                                                                                                                    |
| project_id                | d88c6df92a0a4020915c595032bcd379                                                                                                                                        |
| provider:network_type     | None                                                                                                                                                                    |
| provider:physical_network | None                                                                                                                                                                    |
| provider:segmentation_id  | None                                                                                                                                                                    |
| qos_policy_id             | None                                                                                                                                                                    |
| revision_number           | 1                                                                                                                                                                       |
| router:external           | Internal                                                                                                                                                                |
| segments                  | None                                                                                                                                                                    |
| shared                    | False                                                                                                                                                                   |
| status                    | ACTIVE                                                                                                                                                                  |
| subnets                   |                                                                                                                                                                         |
| tags                      |                                                                                                                                                                         |
| updated_at                | 2021-10-10T10:57:03Z                                                                                                                                                    |
+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

You should see that there is now a public and private network in your project:

```
(user1) [stack@undercloud ~]$ openstack network list
+--------------------------------------+---------+--------------------------------------+
| ID                                   | Name    | Subnets                              |
+--------------------------------------+---------+--------------------------------------+
| 019f687e-667e-4db2-9403-aa2a5b12dd51 | public  | b3af55a2-6f0a-4b23-b5f8-ff111adbbc76 |
| 6c1c797c-0dfa-449c-b20f-974a0b8e71bc | private |                                      |
+--------------------------------------+---------+--------------------------------------+
```

For the network to be useful, we should create a subnet for it:

```
(user1) [stack@undercloud ~]$ openstack subnet create private-subnet \
       --network private \
       --dns-nameserver 8.8.4.4 --gateway 172.16.1.1 \
       --subnet-range 172.16.1.0/24
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field             | Value                                                                                                                                                                   |
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| allocation_pools  | 172.16.1.2-172.16.1.254                                                                                                                                                 |
| cidr              | 172.16.1.0/24                                                                                                                                                           |
| created_at        | 2021-10-10T10:58:58Z                                                                                                                                                    |
| description       |                                                                                                                                                                         |
| dns_nameservers   | 8.8.4.4                                                                                                                                                                 |
| enable_dhcp       | True                                                                                                                                                                    |
| gateway_ip        | 172.16.1.1                                                                                                                                                              |
| host_routes       |                                                                                                                                                                         |
| id                | 6a919130-ad73-4038-8bf5-59dd93801946                                                                                                                                    |
| ip_version        | 4                                                                                                                                                                       |
| ipv6_address_mode | None                                                                                                                                                                    |
| ipv6_ra_mode      | None                                                                                                                                                                    |
| location          | cloud='', project.domain_id=, project.domain_name='Default', project.id='d88c6df92a0a4020915c595032bcd379', project.name='user1project', region_name='regionOne', zone= |
| name              | private-subnet                                                                                                                                                          |
| network_id        | 6c1c797c-0dfa-449c-b20f-974a0b8e71bc                                                                                                                                    |
| prefix_length     | None                                                                                                                                                                    |
| project_id        | d88c6df92a0a4020915c595032bcd379                                                                                                                                        |
| revision_number   | 0                                                                                                                                                                       |
| segment_id        | None                                                                                                                                                                    |
| service_types     |                                                                                                                                                                         |
| subnetpool_id     | None                                                                                                                                                                    |
| tags              |                                                                                                                                                                         |
| updated_at        | 2021-10-10T10:58:58Z                                                                                                                                                    |
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Finally, let's create a <i>router</i> to route traffic from the public to private networks, and vice-versa:

```
(user1) [stack@undercloud ~]$ openstack router create router1
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                                                                   |
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up          | UP                                                                                                                                                                      |
| availability_zone_hints |                                                                                                                                                                         |
| availability_zones      |                                                                                                                                                                         |
| created_at              | 2021-10-10T11:06:12Z                                                                                                                                                    |
| description             |                                                                                                                                                                         |
| external_gateway_info   | null                                                                                                                                                                    |
| flavor_id               | None                                                                                                                                                                    |
| id                      | f3535f6d-0bfe-435a-939f-a60374347517                                                                                                                                    |
| location                | cloud='', project.domain_id=, project.domain_name='Default', project.id='d88c6df92a0a4020915c595032bcd379', project.name='user1project', region_name='regionOne', zone= |
| name                    | router1                                                                                                                                                                 |
| project_id              | d88c6df92a0a4020915c595032bcd379                                                                                                                                        |
| revision_number         | 1                                                                                                                                                                       |
| routes                  |                                                                                                                                                                         |
| status                  | ACTIVE                                                                                                                                                                  |
| tags                    |                                                                                                                                                                         |
| updated_at              | 2021-10-10T11:06:12Z                                                                                                                                                    |
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

```
(user1) [stack@undercloud ~]$ openstack router add subnet router1 private-subnet
(user1) [stack@undercloud ~]$ openstack router set router1 --external-gateway public
```

We're done with the network components.  In the next exercise we'll create an <i>instance</i> (or virtual machine) to run on the network.

[Previous Exercise](exercise01) / [Next Exercise](exercise03)

