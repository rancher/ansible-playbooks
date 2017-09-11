# Installation Instructions

This will walk through the installation of a Rancher environment with the following configuration:
  * two Rancher Servers
  * one HAProxy load balancer
  * one or more nodes

It assumes that you have already done the following things:

  1. Install Prerequisites
    * Ansible
    * Provider-specific requirements (EC2, GCE, etc)
    * External database (RDS, Galera, etc)
  2. Get a certificate
    * You can use a self-signed certificate with Rancher Server without any issues. If you plan to use Kubernetes, you will have to make additional changes for `kubectl` to run. If you can get a real certificate, life will be easier.
  3. Set up your hosts
    * The hosts can be static or dynamic. See the corresponding docs for
    using dynamic inventory with cloud providers.
  
If you have not done these things, please refer back to the README for links to documentation on how to set up these pieces.

## Edit `ansible.cfg`

Set the `ansible_user` variable for the user account you will be using on the remote hosts. For default EC2 hosts, this will be one of the following:

  - Debian: admin
  - Ubuntu: ubuntu

## Edit `group_vars`

  1. Edit `all.yml` to set the Docker version. This is used in [this install command](http://docs.rancher.com/rancher/v1.6/en/hosts/#supported-docker-versions), so versions should only be in the form of `major.minor`.
  2. If you know your registration URL (the final location of your Rancher
  server), set it in `server_url` in `all.yml`.
  2. Edit `server.yml` and set variables as appropriate.

## Edit the Vault

Edit the Vault file with the following:
```
$ ansible-vault edit vars/private.yml
```

Hopefully you changed the password (see [README.md](README.md)), but if not,
the default password is `ansible`.

Once inside, edit the following:
  1. Set the database password.
  2. Set the database admin password
  3. Paste your certificate and key
    * If your certificate has a chain, append the chain to the certificate and place it all in the `cert` section.

## Install The Server

If this is your first run, you have to bring up the server and any load balancers first. Once the server is up, you'll be able to create an API key for the environment and then use it to bring up nodes. 

  ```
  $ ansible-playbook --limit server,loadbalancer rancher.yml
  ```

After the playbook run completes, you can connect to your Rancher Server on 
the public address for your load balancer node. 

If you brought up a single Rancher server, HAProxy was installed on the server itself.

# Automaticallly Adding Nodes

Once the Rancher Server is up, you can automatically register nodes from 
these playbooks. It is not possible to do this prior to the server launching 
because you need to create an API key. Once you have an API key for an 
environment, you can spin up and auto-join hosts at will. 

1. Create an API key pair for the environment. You can do this by expanding
"Advanced Options" from the API/Keys menu item.

2. Enter the key and secret key into the array in the Vault. The array key
should be the unique identifier for the environment. The Default environment
is always `1a5` when Rancher comes up. If you are registering hosts in a
different environment, you can find the environment identifier in the URL
when you have selected that environment in the UI.

    ```
    api_keys:
      1a5:
        access_key: 897BAE0A2E18EC8E1DE3
        secret_key: TFy3kQcE4VM7pqkXgC83E3nN2pthWMTXzPUfjnS4
    ```

3. Tag your nodes with a key of `rancher_env` and the unique identifier for 
the environment.

4. Run the `rancher.yml` playbook with `--tags node` to execute just the node
functions. (*NOTE:* If the playbook doesn't run after you change the `rancher_env` tag, it's because the dynamic inventory plugin caches output for an hour. Clear the cache by running `contrib/inventory/ec2.py --refresh-cache` and then try again.)

    ```
    $ ansible-playbook --limit node --tags node rancher.yml
    ```

These functions will attempt to retrieve the registration URL. If no URL
exists, or if all URLs are inactive, it will register a new URL and use
that URL to register the hosts.

The node functions will not run if any of the following are false:

* No API keys have been defined in the vault
* There is no tag for `rancher_env` defined for the instance
* The value of the `rancher_env` tag is not present in the API keys in the vault

  
