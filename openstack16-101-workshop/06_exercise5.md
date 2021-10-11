---
layout: page
permalink: /openstack16-101-workshop/exercise05
---
__Exercise 5 - Create a working template with load-balanced workload__

Up till now, we've been creating everything in OpenStack by hand. In the real world, this is way too tedious and is prone to human error.

The better way to provision resources is via *Heat Templates*.

Create the file called ```/home/stack/userX/heat_example_basic.yaml``` with the following contents (you can use ``vi``):

```
heat_template_version: rocky

description: >
  Provision a instance with a floating ip

# Set parameters for input and stack provision time
parameters:
  FlavorSize:
    description: The flavor required for the instance
    type: string
    default: "m1.nano"
  TemplateName:
    description: The name of an image to deploy
    type: string
    default: "cirros"
  PrivateNet:
    type: string
    description: Private Network (From openstack network list)
  PrivateSubnet:
    type: string
    description: Private Subnet (From openstack subnet list)
  PublicNet:
    type: string
    description: Public Network (From openstack network list)
  SecurityGroup:
    type: string
    description: Security group
    default: "default"
resources:
  # Create an Instance
  instance0:
    type: OS::Nova::Server
    properties:
      flavor: { get_param: FlavorSize }
      image: { get_param: TemplateName }
      networks:
        - port: { get_resource: instance0_port0 }
      user_data_format: RAW
      user_data: {get_file: /home/stack/user1/webserver.sh}

  # Create a Network Port for that instance
  instance0_port0:
    type: OS::Neutron::Port
    properties:
      network: { get_param: PrivateNet }
      fixed_ips:
        - subnet: { get_param: PrivateSubnet }
      security_groups:
        - { get_param: SecurityGroup }

  # Associate a floating IP to that instance
  instance0_public:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: { get_param: PublicNet }
      port_id: { get_resource: instance0_port0 }


outputs:
  # Output our fixed and floating IPs - This will be shown in 'openstack stack output list'
  instance0_private_ip:
    description: IP address of instance0 in private network
    value: { get_attr: [ instance0, first_address ] }
  instance0_public_ip:
    description: Floating IP address of instance0 in public network
    value: { get_attr: [ instance0_public, floating_ip_address ] }
```

If you're not a fan of ```vi```, then we have pre-prepared a copy of this file.  Copy it from here: ```/home/stack/setup/lab/heat_example_basic.yaml```.

You can get a rough idea from reading the YAML file above that it creates an instance similar to our previous web instance, and allocates a floating ip.  Let's run it:

```
(user1) [stack@undercloud ~]$ openstack stack create example1 \
     -t ~/user1/heat_example_basic.yaml \
     --parameter PrivateNet=private \
     --parameter PrivateSubnet=private-subnet \
     --parameter PublicNet=public
+---------------------+-----------------------------------------+
| Field               | Value                                   |
+---------------------+-----------------------------------------+
| id                  | 7a30528f-ea5d-45fc-b4bb-0c2de2e375f5    |
| stack_name          | example1                                |
| description         | Provision a instance with a floating ip |
|                     |                                         |
| creation_time       | 2021-10-10T12:10:07Z                    |
| updated_time        | None                                    |
| stack_status        | CREATE_IN_PROGRESS                      |
| stack_status_reason | Stack CREATE started                    |
+---------------------+-----------------------------------------+
```

The *stack* is created behind the scenes.  You can check on its progress:

```
(user1) [stack@undercloud ~]$ openstack stack list
+--------------------------------------+------------+-----------------+----------------------+--------------+
| ID                                   | Stack Name | Stack Status    | Creation Time        | Updated Time |
+--------------------------------------+------------+-----------------+----------------------+--------------+
| db483413-ac6e-4eb1-98d9-93c1e9d47ead | example1   | CREATE_COMPLETE | 2021-10-11T04:12:29Z | None         |
+--------------------------------------+------------+-----------------+----------------------+--------------+
```

After a few seconds, you should see all the resources created:

```
(user1) [stack@undercloud ~]$ openstack server list
+--------------------------------------+---------------------------------+--------+----------------------------------+--------+--------+
| ID                                   | Name                            | Status | Networks                         | Image  | Flavor |
+--------------------------------------+---------------------------------+--------+----------------------------------+--------+--------+
| 65a4f35b-8cd3-4d3b-9a0b-c3d6f8070631 | example1-instance0-jgnavudgrvok | ACTIVE | private=172.16.1.104, 10.0.0.171 | cirros |        |
| 07bfac2b-d2f6-485a-a1af-8a663ebc3501 | web-instance                    | ACTIVE | private=172.16.1.4, 10.0.0.162   | cirros |        |
| 31ffca44-9b62-4a35-a1cf-641ba8017b03 | test-instance                   | ACTIVE | private=172.16.1.138, 10.0.0.119 | cirros |        |
+--------------------------------------+---------------------------------+--------+----------------------------------+--------+--------+
```

And you can fetch its web traffic from the *public* (floating ip) allocated to the ```example1-instance0-xxx``` instance:

```
(user1) [stack@undercloud ~]$ curl 10.0.0.171
Welcome to 172.16.1.104
```

Notice that because we did not give this instance a name in the template, OpenStack created a random name for us.

#### Creating a Load Balanced instance

For our final lab exercise, we'll create a load balanced instance.  Create another heat template ```/home/stack/userX/heat_example_lbaas.yaml```:

```
heat_template_version: rocky

description: >
  Provision a instance with a floating ip

# Set parameters for input and stack provision time
parameters:
  FlavorSize:
    description: The flavor required for the instance
    type: string
    default: "m1.nano"
  TemplateName:
    description: The name of an image to deploy
    type: string
    default: "cirros"
  SecurityGroup:
    type: string
    description: Security group
    default: "default"
  router_name:
    label: Router name
    description: Router name with SourceNAT activate
    default: front_router
    type: string
  public_net:
    description: Public Network ID
    type: string
  private_network_net:
    label: Private network address
    default: 172.16.2.0/24
    description: Private network address (e.g. 192.168.0.0/24)
    type: string
  private_network_subnet_low:
    label: Subnet lower IP range
    default: 172.16.2.10
    description: Lower IP address for the private subnet (e.g. 192.168.0.2/24) By default, x.x.x.1 will be set for the gateway
    type: string
  private_network_subnet_high:
    label: Subnet higher IP range
    default: 172.16.2.100
    description: Higher IP address for the private subnet (e.g. 192.168.0.10/24) By default, x.x.x.1 will be set for the gateway
    type: string

resources:
  #-------------------------#
  # Server nodes properties #
  #-------------------------#
  front-node-1:
    type: OS::Nova::Server
    properties:
      name: front-node-1
      flavor: { get_param: FlavorSize }
      image: { get_param: TemplateName }
      security_groups: [ { get_resource: vip_security_group } ]
      networks:
        - network: { get_resource: private_network }
      user_data_format: RAW
      user_data: {get_file: /home/stack/user1/webserver.sh}

  front-node-2:
    type: OS::Nova::Server
    properties:
      name: front-node-2
      flavor: { get_param: FlavorSize }
      image: { get_param: TemplateName }
      security_groups: [ { get_resource: vip_security_group } ]
      networks:
        - network: { get_resource: private_network }
      user_data_format: RAW
      user_data: {get_file: /home/stack/user1/webserver.sh}

  #--------------------#
  # Network properties #
  #--------------------#
  private_network:
    type: OS::Neutron::Net
    properties:
      admin_state_up: true
      name: front-net
      shared: false

  private_subnet:
    type: OS::Neutron::Subnet
    properties:
      allocation_pools:
      - end: { get_param: private_network_subnet_high }
        start: { get_param: private_network_subnet_low }
      cidr: { get_param: private_network_net }
      dns_nameservers: []
      enable_dhcp: true
      host_routes: []
      ip_version: 4
      name: front-subnet
      network_id: { get_resource: private_network }

  vip_security_group:
    type: OS::Neutron::SecurityGroup
    properties:
      description: 'Security group for ICMP, HTTP and SSH'
      name: vip-sec-group
      rules:
      - direction: egress
        ethertype: IPv4
        remote_ip_prefix: 0.0.0.0/0
      - direction: ingress
        protocol: icmp
      - direction: ingress
        ethertype: IPv4
        port_range_max: 80
        port_range_min: 80
        protocol: tcp
      - direction: ingress
        ethertype: IPv4
        port_range_max: 22
        port_range_min: 22
        protocol: tcp


  #----------------------#
  # Router for SourceNAT #
  #----------------------#
  router:
    type: OS::Neutron::Router
    properties:
      name: { get_param: router_name }
      external_gateway_info: { network: { get_param: public_net } }

  router_interface:
    type: OS::Neutron::RouterInterface
    properties:
      router_id: { get_resource : router }
      subnet_id: { get_resource : private_subnet }

  
  #############################
  ### NOTE:  The Load balancer extensions are not installed in Heat in the lab environment.  The following lines are
  ###        included to show how it would look in your environment.
  #############################

  #--------------------------#
  # Load Balancer properties #
  #--------------------------#
  #lb_vip_port:
  #  type: OS::Neutron::Port
  #  properties:
  #    security_groups: [{ get_resource: vip_security_group }]
  #    network_id: { get_resource: private_network }
  #    fixed_ips:
  #      - subnet_id: { get_resource: private_subnet }
  #
  #lb_vip_floating_ip:
  #  type: OS::Neutron::FloatingIP
  #  properties:
  #    floating_network_id: { get_param: public_net }
  #    port_id: { get_resource: lb_vip_port }
  #
  #lb_pool_vip:
  #  type: OS::Neutron::FloatingIPAssociation
  #  properties:
  #    floatingip_id: { get_resource: lb_vip_floating_ip }
  #    #port_id: { 'Fn::Select': ['port_id', {get_attr: [pool, vip] } ] }
  #    port_id: { get_attr: [ pool, vip, port_id ] }
  #
  #monitor:
  #  type: OS::Neutron::HealthMonitor
  #  properties:
  #    type: HTTP
  #    delay: 15
  #    max_retries: 5
  #    timeout: 10
  #
  #pool:
  #  type: OS::Neutron::LBaaS::Pool
  #  properties:
  #    name: lb_front_pool
  #    protocol: HTTP
  #    subnet_id: { get_resource: private_subnet }
  #    lb_method: ROUND_ROBIN
  #    #monitors: [ { get_resource: monitor } ]
  #    vip:
  #      name: front_vip
  #      description: Front-end virtual IP (VIP)
  #      protocol_port: 80
  #      #session_persistence:
  #      #  type: SOURCE_IP
  #
  #lbaas:
  #  type: OS::Neutron::LBaaS::LoadBalancer
  #  properties:
  #    members: [ { get_resource: front-node-1 }, { get_resource: front-node-2 } ]
  #    pool_id: { get_resource: pool }
  #    protocol_port: 80

#---------#
# Outputs #
#---------#
#outputs:
#  LBaaS_floating_ip:
#    description: load balancer floating IP address
#    value: { get_attr: [ lb_vip_floating_ip, floating_ip_address ] }
```

Again, if you're struggling with *vi*, you can copy the file from here: ```/home/stack/setup/lab/heat_example_lbaas.yaml```.

This lab environment does not have the load balancer extensions inserted into the Heat engine, so they are commented out from the template, and we'll do those by hand.  In your production environment, you should not have this problem.

This template creates everything from scratch, including the network and router.  It creates a new private network in the range *172.16.2.0/24*.

Let's run it:

```
(user1) [stack@undercloud ~]$ openstack stack create lbaas \
     -t ~/user1/heat_example_lbaas.yaml \
     --parameter public_net=public 
+---------------------+-----------------------------------------+
| Field               | Value                                   |
+---------------------+-----------------------------------------+
| id                  | 3284a4b9-b385-45be-8576-ac85a642441e    |
| stack_name          | lbaas                                   |
| description         | Provision a instance with a floating ip |
|                     |                                         |
| creation_time       | 2021-10-10T12:20:10Z                    |
| updated_time        | None                                    |
| stack_status        | CREATE_IN_PROGRESS                      |
| stack_status_reason | Stack CREATE started                    |
+---------------------+-----------------------------------------+
```

You'll notice that it creates the new network, subnet and router:

```
(user1) [stack@undercloud ~]$ openstack network list
+--------------------------------------+-----------+--------------------------------------+
| ID                                   | Name      | Subnets                              |
+--------------------------------------+-----------+--------------------------------------+
| 019f687e-667e-4db2-9403-aa2a5b12dd51 | public    | b3af55a2-6f0a-4b23-b5f8-ff111adbbc76 |
| 6c1c797c-0dfa-449c-b20f-974a0b8e71bc | private   | 6a919130-ad73-4038-8bf5-59dd93801946 |
| ac10e5c5-e35d-435e-a1be-a52c05f6b89b | front-net | f541e32c-0034-4240-9ffa-577ca916f1e0 |
+--------------------------------------+-----------+--------------------------------------+
(user1) [stack@undercloud ~]$ openstack subnet list
+--------------------------------------+----------------+--------------------------------------+---------------+
| ID                                   | Name           | Network                              | Subnet        |
+--------------------------------------+----------------+--------------------------------------+---------------+
| 6a919130-ad73-4038-8bf5-59dd93801946 | private-subnet | 6c1c797c-0dfa-449c-b20f-974a0b8e71bc | 172.16.1.0/24 |
| f541e32c-0034-4240-9ffa-577ca916f1e0 | front-subnet   | ac10e5c5-e35d-435e-a1be-a52c05f6b89b | 172.16.2.0/24 |
+--------------------------------------+----------------+--------------------------------------+---------------+
(user1) [stack@undercloud ~]$ openstack router list
+--------------------------------------+--------------+--------+-------+----------------------------------+
| ID                                   | Name         | Status | State | Project                          |
+--------------------------------------+--------------+--------+-------+----------------------------------+
| 001d972d-6e9d-4535-bf08-2b458f07cd58 | front_router | ACTIVE | UP    | d88c6df92a0a4020915c595032bcd379 |
| f3535f6d-0bfe-435a-939f-a60374347517 | router1      | ACTIVE | UP    | d88c6df92a0a4020915c595032bcd379 |
+--------------------------------------+--------------+--------+-------+----------------------------------+
```

It also creates the new instances, and they are in the <i>172.16.2.0/24</i> subnet:

```
(user1) [stack@undercloud ~]$ openstack server list
+--------------------------------------+---------------------------------+--------+----------------------------------+--------+--------+
| ID                                   | Name                            | Status | Networks                         | Image  | Flavor |
+--------------------------------------+---------------------------------+--------+----------------------------------+--------+--------+
| 77f6ba7b-371b-44a2-93df-97a5251924d1 | front-node-2                    | ACTIVE | front-net=172.16.2.64            | cirros |        |
| a064bdd0-bd8b-4ab9-bf8e-09e98f459181 | front-node-1                    | ACTIVE | front-net=172.16.2.18            | cirros |        |
| 65a4f35b-8cd3-4d3b-9a0b-c3d6f8070631 | example1-instance0-jgnavudgrvok | ACTIVE | private=172.16.1.104, 10.0.0.171 | cirros |        |
| 07bfac2b-d2f6-485a-a1af-8a663ebc3501 | web-instance                    | ACTIVE | private=172.16.1.4, 10.0.0.162   | cirros |        |
| 31ffca44-9b62-4a35-a1cf-641ba8017b03 | test-instance                   | ACTIVE | private=172.16.1.138, 10.0.0.119 | cirros |        |
+--------------------------------------+---------------------------------+--------+----------------------------------+--------+--------+
```

However, it doesn't create the load balancer in front of the two instances.  So let's create those by hand.  The commands are listed below without their outputs:

```
(user1) [stack@undercloud ~]$ openstack loadbalancer create --name lbweb --vip-subnet-id front-subnet
(user1) [stack@undercloud ~]$ openstack loadbalancer list
(user1) [stack@undercloud ~]$ sleep 60
(user1) [stack@undercloud ~]$ openstack loadbalancer list
(user1) [stack@undercloud ~]$ openstack loadbalancer listener create --name listenerweb --protocol HTTP --protocol-port 80 lbweb
(user1) [stack@undercloud ~]$ openstack loadbalancer pool create --name poolweb --protocol HTTP  --listener listenerweb --lb-algorithm ROUND_ROBIN
(user1) [stack@undercloud ~]$ openstack server list
(user1) [stack@undercloud ~]$ IPWEB01=$(openstack server show front-node-1 -c addresses -f value | cut -d"=" -f2)
(user1) [stack@undercloud ~]$ IPWEB02=$(openstack server show front-node-2 -c addresses -f value | cut -d"=" -f2)
(user1) [stack@undercloud ~]$ echo $IPWEB01 $IPWEB02
(user1) [stack@undercloud ~]$ openstack loadbalancer member create --name web01 --address $IPWEB01 --protocol-port 80 poolweb
(user1) [stack@undercloud ~]$ openstack loadbalancer member create --name web02 --address $IPWEB02 --protocol-port 80 poolweb
```

At this point, the *load balancer* is created, but its IP address is in the private range.  We need to create a floating ip for it just like we did for the instances:

```
(user1) [stack@undercloud ~]$ VIP=$(openstack loadbalancer show lbweb -c vip_address -f value)
(user1) [stack@undercloud ~]$ PORTID=$(openstack port list --fixed-ip ip-address=$VIP -c ID -f value)
(user1) [stack@undercloud ~]$ openstack floating ip create public
(user1) [stack@undercloud ~]$ FIP=$(openstack floating ip list --status DOWN -c "Floating IP Address" -f value)
(user1) [stack@undercloud ~]$ openstack floating ip set --port $PORTID $FIP
```

To test it out, we can *curl* the load balanced address several times to ensure it is round-robbining between the two instances:

```
(user1) [stack@undercloud ~]$ curl $FIP
Welcome to 172.16.2.18
(user1) [stack@undercloud ~]$ curl $FIP
Welcome to 172.16.2.64
(user1) [stack@undercloud ~]$ curl $FIP
Welcome to 172.16.2.18
(user1) [stack@undercloud ~]$ curl $FIP
Welcome to 172.16.2.64
```

#### Conclusion

That concludes our workshop.  In summary, today we have demonstrated:

* Instances (VMs)
* Software-defined networks
* Software-defined routers
* Software-defined firewalls
* Software-defined load balancers
* OpenStack templates (Heat)

We hope this is just the start of your OpenStack journey.  Good luck!

[Previous Exercise](exercise04)
