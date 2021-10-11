---
layout: page
permalink: /openstack16-101-workshop/setup
---
# Environment Setup

The instructor should have already provisioned the cluster.

You will have received either an Etherpad or Google Doc link link from the instructor.  The Etherpad allows you to pick a username and also has a list of important links and passwords.

All instructions from here on use bash environment variables to reference these links and users. Make sure you substitute them with the proper information whenever you come across them - or add them to your environment variables!

#### Bastion host
The bastion host is a Linux VM that is the jump-off point into the OpenStack lab environment.  The lab environment is not publicly exposed, so it is mandatory for you to use the bastion host.


Connect to the bastion host using an ssh client (eg. Putty or the built-in ssh client in Windows Subsystem for Linux)
```bash
ssh -l $USER $BASTION_HOST
```

Once ready, move on to the first exercise where we will create our first components.

[Next Exercise](exercise01)
