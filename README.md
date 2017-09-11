# Ansible Playbooks for Rancher Hosts

This is a series of playbooks designed to quickly bring up a Rancher 
environment.  

This is an overview of how the project behaves. For specific instructions
on installing and using these playbooks with static and dynamic 
inventories, please see [INSTALL.md](INSTALL.md).

**NOTE:** This is a work in progress. As of this writing it will provision 
an Ubuntu 16.04 environment with static hosts or dynamic hosts in EC2.

In the future it will adapt to RHEL/CentOS/Ubuntu/Debian according to 
the system where the playbooks run. It will also grow to support other
providers with dynamic inventory support.

## Prerequisites

### Install Python

Ubuntu 16.04 doesn't come with Python installed by default. You can either 
install it manually after booting the instances, or you can add the following
to `cloud-config`:
```
#!/bin/bash

apt-get -qq update
apt-get -qq -y install python-pip
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

### Ansible Vault

This project uses the [Ansible Vault](http://docs.ansible.com/ansible/playbooks_vault.html)
for storing private information. There is a sample vault provided with this
repository. The password is `ansible` and can be changed by following the
instructions on rekeying located [here](http://docs.ansible.com/ansible/playbooks_vault.html#rekeying-encrypted-files).

If you wish to skip using the Vault and instead store passwords in plaintext
in the various configuration files, you can do so by removing all references to
`private.yml` from the `vars_files` key in any YAML file in the root of the 
project (e.g. `rancher.yml`, `haproxy.yml`, etc.)

Prior to removing this file, copy its variables out to another variable file,
such as `group_vars/all.yml`.

## Inventory

The project uses a mixture of static and dynamic inventory. Static entries
go into `static_server` and `static_node` in `inventory/hosts`. Dynamic
hosts will be brought in and added to their corresponding groups. All hosts
and groups will then be collected into `server:children`, 
`loadbalancer:children`, and `node:children` for processing by the playbooks 
themselves.

### Supported Inventory Systems

* Static
* EC2

### Enabling Dynamic Inventory Systems

Dynamic inventory provider scripts and their configuration files are stored in `inventory_providers`. To activate one or more of them, symlink them to the `inventory` directory:
```
$ cd inventory
$ ln -s ../inventory_providers/ec2.py ec2.py
```

### EC2

See [INSTALL_EC2.md](INSTALL_EC2.md)

## Playbooks

All playbooks are included in `site.yml`. To execute a full run:
```
$ ansible-playbook site.yml
```

**Note: you won't be able to run a server and node install in the first run.
You will need to install the server and then configure API keys in the Vault.**

Optionally, you can filter by one or more roles:
```
$ ansible-playbook --limit node site.yml
```

Individual playbooks can be run as outlined below.

### Rancher

This playbook installs the version of Docker indicated in `group_vars/all.yml`
on hosts with a `role` tag of `server` or `node`. It goes on to install 
Rancher Server on all hosts with `role` set to `server`. If the `role` is set to `node`, and if there are API keys for the environment located in the
Vault, it will register nodes with the Rancher server.

The Rancher configuration in `group_vars/server.yml` designates the
architecture:

  * single node, internal database
  * single node, external database
  * single node, bind-mount database
  * single node, force HA
    * sets external database
    * use this if you want HA and will add additional servers later

Ansible will automatically configure Rancher to use an external database if
any of the following are true:

  * `use_external_db` is `true`
  * `force_ha` is `true`
  * more than one instance has a tag of `role=server`

If Rancher Server will use an external database, set the database parameters
in `group_vars/server.yml` and set the `db_pass` in the Vault. 

Ansible will perform sanity checks and fail if the database parameters are
missing, but it will not test that the parameters are actually correct.

Ansible will create the database and its user if needed.

See [INSTALL.md](INSTALL.md) for more information about automatic 
host registration.

To run the Rancher playbook on its own, execute:
```
$ ansible-playbook rancher.yml
```

### HAProxy

This playbook installs HAProxy on hosts with `role` set to `loadbalancer`, or
if no hosts exist with this tag, it will install HAProxy on hosts with `role`
set to `server`. The latter is  only appropriate for single-server
environments. If you are running Rancher in an HA configuration, create
additional instances tagged with `role=loadbalancer` and change `haproxy.yml`
to run on nodes with this tag.

**NOTE: If you wish to disable HAProxy entirely, set `haproxy_enabled` to `false` in
`vars/default.yml`.**

After installing HAProxy this playbook then configures it for SSL 
termination using the certificate stored in the Vault. The 
certificate provided in the vault is a self-signed certificate for a fake
domain - please replace it with your own certificate.

HAProxy performs pass-through TCP proxying to Rancher Server using the 
Proxy protocol. This absolves us of the need to have HAProxy perform
additional analysis of the content to enable Websockets or GRPC
communication between the server and the nodes.

Ansible will automatically populate `haproxy.cfg` with the internal IPs of 
all Rancher servers (members of the `server` group). Should these IPs change 
(e.g. if servers are added or removed), or if you need to rebuild the 
configuration (such as if you change the certificate), simply re-run this 
playbook:

  ```
  $ ansible-playbook --tags config haproxy.yml
  ```

### Alternative Post-Install Node Setup (optional)

_This section applies if you do not use these playbooks to register your
nodes with Rancher automatically._

Since you already have an Ansible environment that knows your hosts by 
their EC2 tag, you can use this to install the Rancher Agent onto 
your nodes.

After logging into the server and configuring access control, select
your environment and add a node. Copy the command that Rancher gives you
and use it from your Ansible control station:
```
$ ansible node -a "<command>"
```

This will reach out to all of your nodes in parallel and instruct them
to install the agent. Within a few moments you'll see them appear in the
UI.

