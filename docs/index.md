# OpenStack Heat templates

This directory contains templates for automated Open edX deployment on
the OpenStack platform. For these templates to work, the OpenStack
environment must support:

- OpenStack Keystone,
- OpenStack Nova,
- OpenStack Glance,
- OpenStack Cinder,
- OpenStack Heat,
- OpenStack Neutron (including LBaaS).

The automated deployment time for an OpenStack based Open edX
environment is approximately one hour. It only takes a few minutes to
deploy OpenStack, Open edX's Ansible scheme consumes the rest.

The templates support an OpenStack environment running OpenStack
Icehouse (2014.1) or later.


## Prerequisites

Prior to deploying the Heat templates, ensure that you have completed
the following steps:

- Install a full set of OpenStack client libraries on your system. On
  a contemporary Ubuntu system, `apt-get install
  python-openstackclient python-heatclient` will give you the
  packages you need.
- Retrieve your OpenStack Keystone credentials (a Keystone API
  endpoint, a tenant name, a username, and a password).
- Retrieve the Neutron UUID of your external network, that is, the
  network that floating IPs are allocated from (`neutron net-list` or
  `neutron net-show <networkname>`).
- Create a Nova keypair for yourself (`nova keypair-add` or
  `openstack keypair create`).
- Upload an Ubuntu 12.04 image for your tenant into Glance (`glance
  image-create --copy-from
  https://cloud-images.ubuntu.com/precise/current/precise-server-cloudimg-amd64-disk1.img
  --disk-format qcow2 --container-format bare --name
  precise-server-cloudimg-amd64`), or have your cloud administrator
  do the same for you and make the image public.


## Deployment options

You can use the Heat templates to deploy either a single-node or a
multi-node edX environment.

- In a single-node environment, all Open edX services run on the same
  Nova instance. This configuration is recommended for
  proof-of-concept environment, or when you want to quickly spin up an
  edX stack on your OpenStack cloud for testing or evaluation
  purposes.
- In a multi-node environment, Heat deploys a three-node backend
  cluster running MySQL and MongoDB in a high-availability
  configuration. It also deploys a configurable number of application
  servers running all other Open edX services, and puts all
  application servers behind a load balancer.


### Deploying a single-node environment

To deploy a single-node Open edX environment for testing and
evaluation purposes, use the `edx-single-node.yaml` template. You must
set two mandatory parameters when invoking Heat:

- `public_net_id`, which must be set to the UUID of your external
  network;
- `key_name`, which is the name of the Nova keypair that you'll be
  using to log in once the machines have spun up.

```
heat stack-create \
  -f heat-templates/hot/edx-multi-node.yaml \
  -P "public_net_id=<uuid>" \
  -P "key_name=<name>" \
  <stack_name>
```

To verify that the stack has reached the `CREATE_COMPLETE` state, run:

```
heat stack-show <stack_name>
```

Once stack creation is complete, you can use `heat output-show` to
retrieve the IP address of your Open edX host:

```
heat output-show <stack_name> public_ip
ssh ubuntu@<public_ip>
```

Give the node a few minutes to complete its `cloud-init` configuration, which
will install all ansible prerequisites, including the `edx-configuration`
repository (check the contents of `/var/log/cloud-init.log` for details).  Once
`cloud-init` is done, you will be able to start the ansible playbook run from
within `edx-configuration`.  Before you do so, however, you should enable the
"localhost" host variables, which will configure this deployment of Open edX:

```
cd /var/tmp/edx-configuration/playbooks/openstack/host_vars
cp localhost.example localhost
cd ../../
ansible-playbook -i openstack/inventory.ini -c local openstack-single-node.yml
```

As mentioned above, this playbook run may take one hour or more.  After it's
done, log out of the edx node and edit your local /etc/hosts.  If you didn't
change the example `localhost` host variables, enter the following,
substituting `public_ip` for the IP address you obtained above:

```
<public_ip> lms.example.com studio.example.com
```

You should now be able to access both the LMS and Studio using the following
HTTPS URLs, respectively:

* https://lms.example.com
* https://studio.example.com

#### Single node with the hastexo XBlock

If you want to deploy the hastexo XBlock together with Open edX to your single
node, go back to your installed node and:

1. Locate the following variables in
   `/var/tmp/edx-configuration/playbooks/openstack/host_vars/localhost` (which
   you created above) and change them as described:

    ```
    EDXAPP_EXTRA_REQUIREMENTS:
      - name: "git+https://github.com/hastexo/hastexo-xblock.git@master#egg=hastexo-xblock"
    EDXAPP_ADDL_INSTALLED_APPS:
      - 'hastexo'
    ```

2. Add the `gateone` role to `openstack-single-node.yml` and rerun that
   playbook:

    ```
    $ cd /var/tmp/edx-configuration/playbooks
    $ vim openstack-single-node.yaml
     ...
     - certs
     - demo
     - gateone
    $ ansible-playbook -i openstack/inventory.ini -c local openstack-single-node.yaml
    ```


### Deploying a multi-node environment

To deploy a multi-node Open edX environment, use the
`edx-multi-node.yaml` template. You must set three mandatory
parameters when invoking Heat:

- `public_net_id`, which must be set to the UUID of your external
  network;
- `app_count`, which is the number of application servers you want to
  spin up;
- `key_name`, which is the name of the Nova keypair that you'll be
  using to log in once the machines have spun up.

In addition, you must set the name of the stack.

```
heat stack-create \
  -f heat-templates/hot/edx-multi-node.yaml \
  -P "public_net_id=<uuid>" \
  -P "app_count=<num>" \
  -P "key_name=<name>" \
  <stack_name>
```

To verify that the stack has reached the `CREATE_COMPLETE` state, run:

```
heat stack-show <stack_name>
```

Once stack creation is complete, you can use `heat output-show` to
retrieve the IP address of your deployment host:

```
heat output-show <stack_name> deploy_ip
ssh ubuntu@<deploy_ip>
```

Give the deploy node a few minutes to complete its `cloud-init` configuration,
which will install all ansible prerequisites, including the `edx-configuration`
repository (check the contents of `/var/log/cloud-init.log` for details).  Once
`cloud-init` is done, you will be able to start the ansible playbook run from
within `edx-configuration`.  Before you do so, however, you should enable the
default group and host variables, which will configure this deployment of Open
edX to the cluster:

```
cd /var/tmp/edx-configuration/playbooks/openstack/group_vars
for i in all backend_servers app_servers; do cp $i.example $i; done
cd ../host_vars
for i in 111 112 113; do cp 192.168.122.$i.example 192.168.122.$i; done
```

Be sure to run the `inventory.py` dynamic inventory generator, as opposed to
the static `intentory.ini`, meant for single node deployments:

```
cd /var/tmp/edx-configuration/playbooks
ansible-playbook -i openstack/inventory.py openstack-multi-node.yml
```

This playbook run may take one hour or more.  After it's done, log out of the
deploy node and edit your local /etc/hosts.  If you didn't change any of the
example variables, enter the following, substituting `app_ip` for the IP
address of the app server pool you can obtain with the following Heat command:

```
heat output-show <stack_name> app_ip
vim /etc/hosts
---
<app_ip> lms.example.com studio.example.com
```

You should now be able to access both the LMS and Studio using the following
HTTPS URLs, respectively:

* https://lms.example.com
* https://studio.example.com

To deploy additional application servers within a previously deployed
stack, use the `heat stack-update` command to increase the `app_count`
stack parameter:

```
heat stack-update \
  -f heat-templates/hot/edx-multi-node.yaml \
  -P "public_net_id=<uuid>" \
  -P "app_count=<new_num>" \
  -P "key_name=<name>" \
  <stack_name>
```

If you removed app servers, there's nothing else you need to do.

If you added app servers, however, log back into the deploy node, and run the
multi-node playbook again, limiting the run to the app servers and avoiding
database migrations:

```
cd /var/tmp/edx-configuration/playbooks
ansible-playbook -i openstack/inventory.py openstack-multi-node.yml \
  -e "migrate_db=no" \
  --limit app_servers
```

#### Multiple nodes with the hastexo XBlock

If you want to deploy the hastexo XBlock together with Open edX to your multi
node cluster, go back to your deploy node and:

1. Locate the following variables in
   `/var/tmp/edx-configuration/playbooks/openstack/group_vars/all` (which you
   created above) and change them as described:

    ```
    EDXAPP_EXTRA_REQUIREMENTS:
      - name: "git+https://github.com/hastexo/hastexo-xblock.git@master#egg=hastexo-xblock"
    EDXAPP_ADDL_INSTALLED_APPS:
      - 'hastexo'
    ```

2. Add the `gateone` role to `openstack-multi-node.yml` under the `app_servers`
   section (the last one) and rerun that playbook, limitting the run
   appropriately:

    ```
    $ cd /var/tmp/edx-configuration/playbooks
    $ vim openstack-multi-node.yaml
    ...
    - certs
    - demo
    - gateone
    $ ansible-playbook -i openstack/inventory.py openstack-multi-node.yml \
      --limit app_servers
    ```


## Backing up MySQL and MongoDB data on a multi-node stack

In `playbooks/openstack` you'll find a `backup.yml` playbook that will take
consistent Cinder snapshots of the MySQL and MongoDB volumes of backend
instances, and also rotate them to a configurable limit.  It's designed to run
very quickly, in a non-disruptive manner so as not to incur any downtime for
users.

It is meant to be run from the `deploy` node of a multi-node Heat stack, with
OpenStack authentication variables properly set.  See
`playbooks/openstack/group_vars/backend_servers.example` for a list of
variables that must be changed, including the number of snapshots that will be
maintained in rotation for each instance and database.

It is recommended that this playbook be run with:

    --limit <secondary_backend_node>

With the sample multi-node Heat template, it would either be `--limit
192.168.122.112` or `--limit 192.168.122.113`.  It should not be targetted on
the primary backend node (`192.168.122.111`), because in order to get a
consistent snapshot, prior to snapshotting it will stop the MariaDB service and
sync the filesystem.  (MongoDB doesn't require special handling due to the fact
that its journal is stored in the same volume as the database being
snapshotted.)

Note: the snapshot script expects the backend Cinder volume names to follow a
certain pattern: the MySQL volume must contain "mysql_data" in its name, and
the MongoDB one must contain "mongodb_data".  The provided multi-node Heat
template follows this precise pattern.

### Automating the backup

To run the backup playbook on a schedule, create a cron file such as
`/etc/cron.d/backup`.  If you wish to run the backup daily at 01:05 AM, for
example, copy the following lines into the file:

```
# /etc/cron.d/backup: cron entry for backups

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
HOME=/home/ubuntu

05 1 * * * ubuntu cd /var/tmp/edx-configuration/playbooks; ansible-playbook -i openstack/inventory.py openstack/backup.yml --limit 192.168.122.112 2>&1 >> /var/log/backup.log
```

You'll find a sample `backup.cron` in `playbooks/openstack`, alongside the
backup playbook.

You will also need to create an SSH key without a passphrase for the ubuntu
user on the deploy node, and distribute it to the other nodes.   This is so
that Ansible can connect to the backend node without human intervention.  A
hands-free way to create one and copy it to all backend nodes would be:

```
KEYFILE=~/.ssh/id_rsa
ssh-keygen -t rsa -N "" -f $KEYFILE
for i in 192.168.122.111 192.168.122.112 192.168.122.113; do
    ssh-keyscan $i >> ~/.ssh/known_hosts
    ssh-copy-id -i $KEYFILE $i
done
```
