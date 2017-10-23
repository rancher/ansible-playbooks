# Installation Instructions

This will walk through the prerequisites to use EC2 dynamic inventory.

## Create Security Group

Create a security group called `rancher` with the following ports open. 

  * HTTPS from anywhere
  * SSH from anywhere
  * UDP/500 from anywhere
  * UDP/4500 from anywhere

Save the group so that it receives an identifier, and then edit it and add the following ports, setting them up to only be accessible from the security group itself:

  * TCP/8080
  * TCP/2376
  * TCP/9345
  * TCP/3306
  * ALL IPv4 ICMP

This setup will allow any host in the security group to communicate with other hosts and with the Rancher Server.

Once you launch containers on the nodes, you may need to open additional ports for those services or add the nodes to a security group that contains those open ports.

## Launch Instances

*NOTE:* All instances must run Ubuntu 16.04 or these playbooks may not work correctly.

Launch the following instances in a VPC of your choice. While they don't 
_have_ to be in the same AZ, placing them in the same AZ will reduce
AWS charges for inter-AZ traffic. The configuration expects that nodes will 
all be able to reach one another on the private IPs. 

Add the following code  to the "User data" section under "Advanced Details" in Step 3:
```
#!/bin/bash
apt-get -qq update && apt-get -qq install python python-pip
```

Alternatively, add the following:
```
#cloud-config

package_upgrade: true
package_update: true
packages: 
  - curl
  - python
  - python-pip
```

  * Rancher Server
    * 1 or 2 `t2.small` instances with 16 GB of disk
    * tag these hosts with
      * `project=rancher`
      * `rancher_role=server`
  * Load Balancer
    * 1 `t2.nano` instance with 8 GB of disk
    * tag these hosts with
      * `project=rancher`
      * `rancher_role=loadbalancer`
  * Nodes
    * 1 or more `t2.large` instances with 32 GB of disk
    * tag these hosts with
      * `project=rancher`
      * `rancher_role=node`
      * `rancher_env=1a5` (to put them in the Default environment)
  * Database
    * 1 `db.t2.small` RDS instance (MySQL or compatible)

Place all instances in the `rancher` security group.

*NOTE:* You may skip starting the RDS instance if you set `use_external_db`
to `false` and only spin up a single instance for the Rancher Server. Ansible
will then launch the Server container without pointing to an external
database.

## Enable the EC2 inventory provider

In order to enable the provider, symlink it from `inventory_providers` to 
`inventory`:

  ```
  $ cd inventory
  $ ln -s ../inventory_providers/ec2.py ec2.py
  ```

## Edit `ec2.ini`

The filter set in `inventory_providers/ec2.ini` will collect all host with
`project=rancher` and then act on them according to their `role` tag. If you
used something other than `project=rancher` (e.g. `owner=adrian`), edit `ec2.ini` and change the filter set on/around line 159. 

Additionally, if you're working in regions other than `us-east-1` and 
`us-west-2`, set your regions in line 14.

## Test the connection

If everything is set up correctly, you can populate the cache and verify that
the output is as expected:

  ```
  $ cd inventory
  $ ./ec2.py --refresh-cache | less
  ```

The cache is set to 1h by default. If you make changes to EC2 tags or other
parameters, you'll need to refresh the cache before Ansible picks up the 
changes.

## Continue the installation

Continue the installation with [INSTALL.md](INSTALL.md).

  
