---
layout: page
permalink: /openstack16-101-workshop/exercise03
---
__Exercise 3 -  Create a basic instance (Virtual Machine)__

There's a few things we must set up before we can run an *instance*.  The first of these is to create a base OS image.

We have already downloaded a base image for you, which runs the CirrOS Linux instance.  CirrOS is a minimal Linux distribution designed to test clouds.  But it's functional enough for this lab.

You will find the image here:

```
(user1) [stack@undercloud ~]$ ls ~/cir*
/home/stack/cirros-0.4.0-x86_64-disk.img
```

We must ingest the image into *glance*, which is OpenStack's image store:

```
(user1) [stack@undercloud ~]$ openstack image create cirros \
       --file ~/cirros-0.4.0-x86_64-disk.img \
       --disk-format qcow2 --container-format bare
+------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                                                                                                                                                                                                              |
+------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                                                                                                                                                                                                                                                                   |
| container_format | bare                                                                                                                                                                                                                                                                                               |
| created_at       | 2021-10-10T11:15:52Z                                                                                                                                                                                                                                                                               |
| disk_format      | qcow2                                                                                                                                                                                                                                                                                              |
| file             | /v2/images/2a41ae1c-753a-41c5-85a8-e5ea6586672f/file                                                                                                                                                                                                                                               |
| id               | 2a41ae1c-753a-41c5-85a8-e5ea6586672f                                                                                                                                                                                                                                                               |
| min_disk         | 0                                                                                                                                                                                                                                                                                                  |
| min_ram          | 0                                                                                                                                                                                                                                                                                                  |
| name             | cirros                                                                                                                                                                                                                                                                                             |
| owner            | d88c6df92a0a4020915c595032bcd379                                                                                                                                                                                                                                                                   |
| properties       | direct_url='swift+config://ref1/glance/2a41ae1c-753a-41c5-85a8-e5ea6586672f', os_hash_algo='sha512', os_hash_value='6513f21e44aa3da349f248188a44bc304a3653a04122d8fb4535423c8e1d14cd6a153f735bb0982e2161b5b5186106570c17a9e58b64dd39390617cd5a350f78', os_hidden='False', stores='default_backend' |
| protected        | False                                                                                                                                                                                                                                                                                              |
| schema           | /v2/schemas/image                                                                                                                                                                                                                                                                                  |
| size             | 12716032                                                                                                                                                                                                                                                                                           |
| status           | active                                                                                                                                                                                                                                                                                             |
| tags             |                                                                                                                                                                                                                                                                                                    |
| updated_at       | 2021-10-10T11:15:53Z                                                                                                                                                                                                                                                                               |
| virtual_size     | None                                                                                                                                                                                                                                                                                               |
| visibility       | shared                                                                                                                                                                                                                                                                                             |
+------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

You can list the OS images in your project in OpenStack:

```
(user1) [stack@undercloud ~]$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 2a41ae1c-753a-41c5-85a8-e5ea6586672f | cirros | active |
+--------------------------------------+--------+--------+
```

It is possible to create *public* images.  An administrator can create public images for consumption by all users of the OpenStack Cluster.  Simply add the *--public* parameter when creating the image, and it should be visible to all tenancies.

The final dependency before we create an instance is to open up some ports in the security group that controls network access in this project.  Every project gets a *default* security group, and we'll use that.

```
(user1) [stack@undercloud ~]$ openstack security group list
+--------------------------------------+---------+------------------------+----------------------------------+------+
| ID                                   | Name    | Description            | Project                          | Tags |
+--------------------------------------+---------+------------------------+----------------------------------+------+
| a76affe4-3137-4aa5-816c-7ae832c8bf31 | default | Default security group | d88c6df92a0a4020915c595032bcd379 | []   |
+--------------------------------------+---------+------------------------+----------------------------------+------+
```

Of course, you can create other security groups to provide different levels of access to different instances.  But for now, we'll modify the default by adding two firewall rules to it.  This will open *icmp* and *port 22*:

```
(user1) [stack@undercloud ~]$ openstack security group rule create --proto icmp default
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field             | Value                                                                                                                                                                   |
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| created_at        | 2021-10-10T11:23:23Z                                                                                                                                                    |
| description       |                                                                                                                                                                         |
| direction         | ingress                                                                                                                                                                 |
| ether_type        | IPv4                                                                                                                                                                    |
| id                | 6a81fb02-c79b-4914-8ce1-0a375859ff18                                                                                                                                    |
| location          | cloud='', project.domain_id=, project.domain_name='Default', project.id='d88c6df92a0a4020915c595032bcd379', project.name='user1project', region_name='regionOne', zone= |
| name              | None                                                                                                                                                                    |
| port_range_max    | None                                                                                                                                                                    |
| port_range_min    | None                                                                                                                                                                    |
| project_id        | d88c6df92a0a4020915c595032bcd379                                                                                                                                        |
| protocol          | icmp                                                                                                                                                                    |
| remote_group_id   | None                                                                                                                                                                    |
| remote_ip_prefix  | 0.0.0.0/0                                                                                                                                                               |
| revision_number   | 0                                                                                                                                                                       |
| security_group_id | a76affe4-3137-4aa5-816c-7ae832c8bf31                                                                                                                                    |
| tags              | []                                                                                                                                                                      |
| updated_at        | 2021-10-10T11:23:23Z                                                                                                                                                    |
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

```
(user1) [stack@undercloud ~]$ openstack security group rule create --dst-port 22 --proto tcp default
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field             | Value                                                                                                                                                                   |
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| created_at        | 2021-10-10T11:23:28Z                                                                                                                                                    |
| description       |                                                                                                                                                                         |
| direction         | ingress                                                                                                                                                                 |
| ether_type        | IPv4                                                                                                                                                                    |
| id                | 4cce05fd-f7cc-46dc-8765-449deb581d94                                                                                                                                    |
| location          | cloud='', project.domain_id=, project.domain_name='Default', project.id='d88c6df92a0a4020915c595032bcd379', project.name='user1project', region_name='regionOne', zone= |
| name              | None                                                                                                                                                                    |
| port_range_max    | 22                                                                                                                                                                      |
| port_range_min    | 22                                                                                                                                                                      |
| project_id        | d88c6df92a0a4020915c595032bcd379                                                                                                                                        |
| protocol          | tcp                                                                                                                                                                     |
| remote_group_id   | None                                                                                                                                                                    |
| remote_ip_prefix  | 0.0.0.0/0                                                                                                                                                               |
| revision_number   | 0                                                                                                                                                                       |
| security_group_id | a76affe4-3137-4aa5-816c-7ae832c8bf31                                                                                                                                    |
| tags              | []                                                                                                                                                                      |
| updated_at        | 2021-10-10T11:23:28Z                                                                                                                                                    |
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

We're now ready to create our instance:

```
(user1) [stack@undercloud ~]$ openstack server create test-instance --flavor m1.nano --image cirros --network private
+-----------------------------+------------------------------------------------------------------------------------+
| Field                       | Value                                                                              |
+-----------------------------+------------------------------------------------------------------------------------+
| OS-DCF:diskConfig           | MANUAL                                                                             |
| OS-EXT-AZ:availability_zone |                                                                                    |
| OS-EXT-STS:power_state      | NOSTATE                                                                            |
| OS-EXT-STS:task_state       | scheduling                                                                         |
| OS-EXT-STS:vm_state         | building                                                                           |
| OS-SRV-USG:launched_at      | None                                                                               |
| OS-SRV-USG:terminated_at    | None                                                                               |
| accessIPv4                  |                                                                                    |
| accessIPv6                  |                                                                                    |
| addresses                   |                                                                                    |
| adminPass                   | s2vEAke7j6wQ                                                                       |
| config_drive                |                                                                                    |
| created                     | 2021-10-10T11:25:53Z                                                               |
| description                 | None                                                                               |
| flavor                      | disk='1', ephemeral='0', , original_name='m1.nano', ram='128', swap='0', vcpus='1' |
| hostId                      |                                                                                    |
| id                          | 31ffca44-9b62-4a35-a1cf-641ba8017b03                                               |
| image                       | cirros (2a41ae1c-753a-41c5-85a8-e5ea6586672f)                                      |
| key_name                    | None                                                                               |
| locked                      | False                                                                              |
| locked_reason               | None                                                                               |
| name                        | test-instance                                                                      |
| progress                    | 0                                                                                  |
| project_id                  | d88c6df92a0a4020915c595032bcd379                                                   |
| properties                  |                                                                                    |
| security_groups             | name='default'                                                                     |
| server_groups               | []                                                                                 |
| status                      | BUILD                                                                              |
| tags                        | []                                                                                 |
| trusted_image_certificates  | None                                                                               |
| updated                     | 2021-10-10T11:25:53Z                                                               |
| user_id                     | 4f8d833b3b074a6b86f1623c532f57f5                                                   |
| volumes_attached            |                                                                                    |
+-----------------------------+------------------------------------------------------------------------------------+
```

The command returns quite quickly, as it doesn't wait for the instance to come up.  You can check on its status:

```
(user1) [stack@undercloud ~]$ openstack server list
+--------------------------------------+---------------+--------+----------+--------+--------+
| ID                                   | Name          | Status | Networks | Image  | Flavor |
+--------------------------------------+---------------+--------+----------+--------+--------+
| 31ffca44-9b62-4a35-a1cf-641ba8017b03 | test-instance | BUILD  |          | cirros |        |
+--------------------------------------+---------------+--------+----------+--------+--------+

(user1) [stack@undercloud ~]$ openstack server list
+--------------------------------------+---------------+--------+----------------------+--------+--------+
| ID                                   | Name          | Status | Networks             | Image  | Flavor |
+--------------------------------------+---------------+--------+----------------------+--------+--------+
| 31ffca44-9b62-4a35-a1cf-641ba8017b03 | test-instance | ACTIVE | private=172.16.1.138 | cirros |        |
+--------------------------------------+---------------+--------+----------------------+--------+--------+
```

Once it comes up, you'll notice that the IP address allocate to the instance is indeed on the private network (in the *172.16.1.X* range).

You can view the server console log for the instance:

```
(user1) [stack@undercloud ~]$ openstack console log show test-instance
[    0.000000] Initializing cgroup subsys cpuset
[    0.000000] Initializing cgroup subsys cpu
[    0.000000] Initializing cgroup subsys cpuacct
[    0.000000] Linux version 4.4.0-28-generic (buildd@lcy01-13) (gcc version 5.3.1 20160413 (Ubuntu 5.3.1-14ubuntu2.1) ) #47-Ubuntu SMP Fri Jun 24 10:09:13 UTC 2016 (Ubuntu 4.4.0-28.47-generic 4.4.13)
[    0.000000] Command line: LABEL=cirros-rootfs ro console=tty1 console=ttyS0
[    0.000000] KERNEL supported cpus:
[    0.000000]   Intel GenuineIntel
[    0.000000]   AMD AuthenticAMD
[    0.000000]   Centaur CentaurHauls
...
=== datasource: ec2 net ===
instance-id: i-00000001
name: N/A
availability-zone: nova
local-hostname: test-instance
launch-index: 0
=== cirros: current=0.4.0 latest=0.5.2 uptime=5.49 ===
  ____               ____  ____
 / __/ __ ____ ____ / __ \/ __/
/ /__ / // __// __// /_/ /\ \ 
\___//_//_/  /_/   \____/___/ 
   http://cirros-cloud.net


login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
/dev/root resized successfully [took 0.45s]
test-instance login: 
```

But you will notice that we have no network connectivity to the server:

```
(user1) [stack@undercloud ~]$ ping -c3 172.16.1.138
PING 172.16.1.138 (172.16.1.138) 56(84) bytes of data.

--- 172.16.1.138 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 37ms

```

That's because there's no route from our Workstation (on the public network) and the private network.

Let's allocate a public-facing IP address for this instance.  We do this via a *floating ip* on the public network:

```
(user1) [stack@undercloud ~]$ openstack floating ip create public
+---------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field               | Value                                                                                                                                                                                             |
+---------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| created_at          | 2021-10-10T11:34:33Z                                                                                                                                                                              |
| description         |                                                                                                                                                                                                   |
| dns_domain          |                                                                                                                                                                                                   |
| dns_name            |                                                                                                                                                                                                   |
| fixed_ip_address    | None                                                                                                                                                                                              |
| floating_ip_address | 10.0.0.119                                                                                                                                                                                        |
| floating_network_id | 019f687e-667e-4db2-9403-aa2a5b12dd51                                                                                                                                                              |
| id                  | 8b4c77d8-2a1e-46e8-b311-3b4225561716                                                                                                                                                              |
| location            | Munch({'cloud': '', 'region_name': 'regionOne', 'zone': None, 'project': Munch({'id': 'd88c6df92a0a4020915c595032bcd379', 'name': 'user1project', 'domain_id': None, 'domain_name': 'Default'})}) |
| name                | 10.0.0.119                                                                                                                                                                                        |
| port_details        | None                                                                                                                                                                                              |
| port_id             | None                                                                                                                                                                                              |
| project_id          | d88c6df92a0a4020915c595032bcd379                                                                                                                                                                  |
| qos_policy_id       | None                                                                                                                                                                                              |
| revision_number     | 0                                                                                                                                                                                                 |
| router_id           | None                                                                                                                                                                                              |
| status              | DOWN                                                                                                                                                                                              |
| subnet_id           | None                                                                                                                                                                                              |
| tags                | []                                                                                                                                                                                                |
| updated_at          | 2021-10-10T11:34:33Z                                                                                                                                                                              |
+---------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

You'll notice that the IP address that was created is in the public range (10 network).  Next we'll allocate the *floating ip* to the instance:

```
(user1) [stack@undercloud ~]$ openstack floating ip list
+--------------------------------------+---------------------+------------------+------+--------------------------------------+----------------------------------+
| ID                                   | Floating IP Address | Fixed IP Address | Port | Floating Network                     | Project                          |
+--------------------------------------+---------------------+------------------+------+--------------------------------------+----------------------------------+
| 8b4c77d8-2a1e-46e8-b311-3b4225561716 | 10.0.0.119          | None             | None | 019f687e-667e-4db2-9403-aa2a5b12dd51 | d88c6df92a0a4020915c595032bcd379 |
+--------------------------------------+---------------------+------------------+------+--------------------------------------+----------------------------------+
(user1) [stack@undercloud ~]$ FIP=$(openstack floating ip list --status DOWN -c "Floating IP Address" -f value)
(user1) [stack@undercloud ~]$ openstack server add floating ip test-instance $FIP
(user1) [stack@undercloud ~]$ openstack server list
+--------------------------------------+---------------+--------+----------------------------------+--------+--------+
| ID                                   | Name          | Status | Networks                         | Image  | Flavor |
+--------------------------------------+---------------+--------+----------------------------------+--------+--------+
| 31ffca44-9b62-4a35-a1cf-641ba8017b03 | test-instance | ACTIVE | private=172.16.1.138, 10.0.0.119 | cirros |        |
+--------------------------------------+---------------+--------+----------------------------------+--------+--------+
```

From the server list above, you can see that the instance is now allocated the floating ip address.  We should be able to ping it and log into it (because we opened up those ports in the security group):

```
(user1) [stack@undercloud ~]$ ping -c3 $FIP
PING 10.0.0.119 (10.0.0.119) 56(84) bytes of data.
64 bytes from 10.0.0.119: icmp_seq=1 ttl=63 time=8.65 ms
64 bytes from 10.0.0.119: icmp_seq=2 ttl=63 time=1.28 ms
64 bytes from 10.0.0.119: icmp_seq=3 ttl=63 time=0.703 ms

--- 10.0.0.119 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 0.703/3.544/8.645/3.614 ms

(user1) [stack@undercloud ~]$ ssh cirros@${FIP}
The authenticity of host '10.0.0.119 (10.0.0.119)' can't be established.
ECDSA key fingerprint is SHA256:Auym0twfjRZuF0NL3k7N3Nrj6uEEcgoTzqddEusQ3lg.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.0.119' (ECDSA) to the list of known hosts.
cirros@10.0.0.119's password: 

$ uname -a
Linux test-instance 4.4.0-28-generic #47-Ubuntu SMP Fri Jun 24 10:09:13 UTC 2016 x86_64 GNU/Linux
```

Once other participants in the workshop have created their instances with floating IP address, you should also be able to ping from within your instance.

In the next exercise, we'll create another instance that is slightly more useful.

[Previous Exercise](exercise02) / [Next Exercise](exercise04)
