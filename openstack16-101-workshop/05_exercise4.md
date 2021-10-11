---
layout: page
permalink: /openstack16-101-workshop/exercise04
---
__Exercise 4 - Create a more functional instance__

In this exercise, we'll create a slight variation on the first instance by have it service some web traffic.

In order to service web traffic, we need to open up port 80 in the security group:

```
(user1) [stack@undercloud ~]$ openstack security group rule create --dst-port 80 --proto tcp default
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field             | Value                                                                                                                                                                   |
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| created_at        | 2021-10-10T11:50:19Z                                                                                                                                                    |
| description       |                                                                                                                                                                         |
| direction         | ingress                                                                                                                                                                 |
| ether_type        | IPv4                                                                                                                                                                    |
| id                | 3ff9d141-1567-4015-a0e4-f27dca3eaa1a                                                                                                                                    |
| location          | cloud='', project.domain_id=, project.domain_name='Default', project.id='d88c6df92a0a4020915c595032bcd379', project.name='user1project', region_name='regionOne', zone= |
| name              | None                                                                                                                                                                    |
| port_range_max    | 80                                                                                                                                                                      |
| port_range_min    | 80                                                                                                                                                                      |
| project_id        | d88c6df92a0a4020915c595032bcd379                                                                                                                                        |
| protocol          | tcp                                                                                                                                                                     |
| remote_group_id   | None                                                                                                                                                                    |
| remote_ip_prefix  | 0.0.0.0/0                                                                                                                                                               |
| revision_number   | 0                                                                                                                                                                       |
| security_group_id | a76affe4-3137-4aa5-816c-7ae832c8bf31                                                                                                                                    |
| tags              | []                                                                                                                                                                      |
| updated_at        | 2021-10-10T11:50:19Z                                                                                                                                                    |
+-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Since CirrOS doesn't run a full-blown web server, we'll cheat a little with creating the traffic:

```
(user1) [stack@undercloud ~]$ cat <<EOF > ~/${OS_USERNAME}/webserver.sh
#!/bin/sh

MYIP=\$(/sbin/ifconfig eth0|grep 'inet addr'|awk -F: '{print \$2}'| awk '{print \$1}');
OUTPUT_STR="Welcome to \$MYIP\r"
OUTPUT_LEN=\${#OUTPUT_STR}

while true; do
    echo -e "HTTP/1.0 200 OK\r\nContent-Length: \${OUTPUT_LEN}\r\n\r\n\${OUTPUT_STR}" | sudo nc -l -p 80
done

EOF
```

We'll create a new instance using this script, and this time we'll wait for the instance to finish starting up before returning:

```
(user1) [stack@undercloud ~]$ openstack server create \
     --image cirros \
     --flavor m1.nano  \
     --user-data ~/${OS_USERNAME}/webserver.sh \
     web-instance  --wait

+-----------------------------+------------------------------------------------------------------------------------+
| Field                       | Value                                                                              |
+-----------------------------+------------------------------------------------------------------------------------+
| OS-DCF:diskConfig           | MANUAL                                                                             |
| OS-EXT-AZ:availability_zone | nova                                                                               |
| OS-EXT-STS:power_state      | Running                                                                            |
| OS-EXT-STS:task_state       | None                                                                               |
| OS-EXT-STS:vm_state         | active                                                                             |
| OS-SRV-USG:launched_at      | 2021-10-10T11:55:07.000000                                                         |
| OS-SRV-USG:terminated_at    | None                                                                               |
| accessIPv4                  |                                                                                    |
| accessIPv6                  |                                                                                    |
| addresses                   | private=172.16.1.4                                                                 |
| adminPass                   | 4SMchMEd6Yd3                                                                       |
| config_drive                |                                                                                    |
| created                     | 2021-10-10T11:54:52Z                                                               |
| description                 | None                                                                               |
| flavor                      | disk='1', ephemeral='0', , original_name='m1.nano', ram='128', swap='0', vcpus='1' |
| hostId                      | 646171d3eceba2c255212732f1b66af43456048a70bc541e868d828c                           |
| id                          | 07bfac2b-d2f6-485a-a1af-8a663ebc3501                                               |
| image                       | cirros (2a41ae1c-753a-41c5-85a8-e5ea6586672f)                                      |
| key_name                    | None                                                                               |
| locked                      | False                                                                              |
| locked_reason               | None                                                                               |
| name                        | web-instance                                                                       |
| progress                    | 0                                                                                  |
| project_id                  | d88c6df92a0a4020915c595032bcd379                                                   |
| properties                  |                                                                                    |
| security_groups             | name='default'                                                                     |
| server_groups               | []                                                                                 |
| status                      | ACTIVE                                                                             |
| tags                        | []                                                                                 |
| trusted_image_certificates  | None                                                                               |
| updated                     | 2021-10-10T11:55:06Z                                                               |
| user_id                     | 4f8d833b3b074a6b86f1623c532f57f5                                                   |
| volumes_attached            |                                                                                    |
+-----------------------------+------------------------------------------------------------------------------------+
```

As before, let's allocate a floating ip address to the instance:

```
(user1) [stack@undercloud ~]$ openstack floating ip create public
(user1) [stack@undercloud ~]$ FIP=$(openstack floating ip list --status DOWN -c "Floating IP Address" -f value)
(user1) [stack@undercloud ~]$ openstack server add floating ip web-instance $FIP
(user1) [stack@undercloud ~]$ openstack server list
+--------------------------------------+---------------+--------+----------------------------------+--------+--------+
| ID                                   | Name          | Status | Networks                         | Image  | Flavor |
+--------------------------------------+---------------+--------+----------------------------------+--------+--------+
| 07bfac2b-d2f6-485a-a1af-8a663ebc3501 | web-instance  | ACTIVE | private=172.16.1.4, 10.0.0.162   | cirros |        |
| 31ffca44-9b62-4a35-a1cf-641ba8017b03 | test-instance | ACTIVE | private=172.16.1.138, 10.0.0.119 | cirros |        |
+--------------------------------------+---------------+--------+----------------------------------+--------+--------+
(user1) [stack@undercloud ~]$ curl $FIP
Welcome to 172.16.1.4

```

Neat.

In the final exercise, we'll extend this use-case to make it highly available, more closely resembling a real-world scenario.

[Previous Exercise](exercise03) / [Next Exercise](exercise05)
