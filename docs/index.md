# OpenStack Heat templates

This directory contains templates for automated Open edX deployment on
the OpenStack platform. For these templates to work, the OpenStack
environment must support:

- OpenStack Keystone,
- OpenStack Nova,
- OpenStack Glance,
- OpenStack Cinder,
- OpenStack Heat,
- OpenStack Neutron (including LBaaS),
- OpenStack Swift, or any drop-in replacement service that understands
  the OpenStack Swift API and is registered as an object service
  endpoint.

The automated deployment time for an OpenStack based Open edX
environment is approximately one hour. It only takes a few minutes to
deploy OpenStack, Open edX's Ansible scheme consumes the rest.

The templates use syntax that supports an OpenStack environment
running OpenStack Juno (2014.2) or later. Please note that it is not
recommended to run Open edX on environments running an OpenStack
release marked end-of-life (EOL); see
[the OpenStack release table](https://releases.openstack.org/) for
details.


## Prerequisites

Prior to deploying the Heat templates, ensure that you have completed
the following steps:

- Install the OpenStack client libraries and command-line interfaces
  (CLIs) on your system. Refer to the relevant section in
  [the OpenStack User Guide](http://docs.openstack.org/user-guide/common/cli-install-openstack-command-line-clients.html)
  for details on doing so, for a variety of operating systems and
  platforms.

- Retrieve your OpenStack Keystone credentials (a Keystone API
  endpoint, a tenant name, a username, and a password).

- Retrieve the Neutron UUID of your external network, that is, the
  network that floating IPs are allocated from:

```
openstack network list
openstack network show <networkname>
```

- Create a Nova keypair for yourself (`openstack keypair create`).
- Upload an Ubuntu 14.04 image for your tenant into Glance:

```
wget https://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --file trusty-server-cloudimg-amd64-disk1.img \
  --name ubuntu-14.04-server-cloudimg
```

Alternatively, you may ask your cloud administrator do the same for
 you, and mark the image public by adding the `--public` option.


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
openstack stack create \
    --template heat-templates/hot/edx-single-node.yaml \
    --parameter public_net_id=<uuid> \
    --parameter key_name=<name> \
    <stack_name>
```

To verify that the stack has reached the `CREATE_COMPLETE` state, run:

```
openstack stack show <stack_name>
```

Once stack creation is complete, you can use `heat output-show` to
retrieve the IP address of your Open edX host:

```
openstack stack output show <stack_name> public_ip
ssh ubuntu@<public_ip>
```

Give the node a few minutes to complete its `cloud-init` configuration, which
will install all ansible prerequisites, including the `edx-configuration`
repository (check the contents of `/var/log/cloud-init.log` for details).  Once
`cloud-init` is done, you will be able to start the ansible playbook run from
within `edx-configuration`.

Before you do so, however, you should enable the "localhost" host variables,
which will configure this deployment of Open edX.  Due to how Ansible variable
precedence works, it is recommended that you copy the sample ones to a separate
directory:

```
cp /var/tmp/edx-configuration/playbooks/openstack /var/tmp/edx-configuration-secrets
cd /var/tmp/edx-configuration-secrets/host_vars
cp localhost.example localhost
cd /var/tmp/edx-configuration/playbooks
ansible-playbook -i ../../edx-configuration-secrets/inventory.ini -c local openstack-single-node.yml
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
   `/var/tmp/edx-configuration-secrets/host_vars/localhost` (which you created
   above) and change them as described.  Set the `os_*` variables to the
   OpenStack cloud of your choice.

```
EDXAPP_EXTRA_REQUIREMENTS:
  - name: "git+https://github.com/hastexo/hastexo-xblock.git@master#egg=hastexo-xblock"
EDXAPP_ADDL_INSTALLED_APPS:
  - 'hastexo'
EDXAPP_XBLOCK_SETTINGS:
  hastexo:
    providers:
      default:
        os_auth_url: ""
        os_auth_token: ""
        os_username: ""
        os_password: ""
        os_user_id: ""
        os_user_domain_id: ""
        os_user_domain_name: ""
        os_project_id: ""
        os_project_name: ""
        os_project_domain_id: ""
        os_project_domain_name: ""
        os_region_name: ""
```

2. Check out the `hastexo/integration/base` branch of edx-configuration, which
   contains the `gateone` role:

```
$ cd /var/tmp/edx-configuration
$ git checkout -b hastexo/integration/base origin/hastexo/integration/base
```

3. Add the `gateone` role to `openstack-single-node.yml` and rerun that
   playbook:

```
$ cd /var/tmp/edx-configuration/playbooks
$ vim openstack-single-node.yaml
 ...
 - certs
 - demo
 - gateone
$ ansible-playbook -i ../../edx-configuration-secrets/inventory.ini -c local openstack-single-node.yaml
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
stack=<stack_name>
openstack stack create \
    --template heat-templates/hot/edx-multi-node.yaml \
    --parameter public_net_id=<uuid> \
    --parameter app_count=<num> \
    --parameter key_name=<name> \
    $stack
```

To verify that the stack has reached the `CREATE_COMPLETE` state, run:

```
openstack stack show $stack
```

Once stack creation is complete, you can use `openstack stack output show` to
retrieve the IP address of your deployment host, and then to connect to it,
making sure to forward your SSH authentication agent:

```
openstack stack output show $stack deploy_ip
ssh ubuntu@<deploy_ip> -A
```

Before you continue, you'll need the `hastexo/integration/base` fork of
`edx-configuration`.  Clone it, and then enable the sample group and host
variables, which will configure this deployment of Open edX.  Due to how
Ansible variable precedence works, it is recommended that you copy the sample
ones to a separate directory:

```
git clone https://github.com/hastexo/edx-configuration.git -b hastexo/integration/base
cp -a edx-configuration/playbooks/openstack edx-configuration-secrets
cd edx-configuration-secrets/group_vars
for i in *.example; do cp $i ${i%.example}; done
cd ../host_vars
for i in 192*.example; do cp $i ${i%.example}; done
```

It is recommended that you use the hastexo edX repositories, and in particular
the `hastexo/integration/hastexo` branch of `edx-platform`.  These are known to
work correctly in an OpenStack cluster.  To do so, edit
`edx-configuration-secrets/group_vars/all`, and replace the repository
variables with the following:

```
# Repos
edx_platform_repo: "https://{{ COMMON_GIT_MIRROR }}/hastexo/edx-platform.git"
edx_platform_version: "hastexo/integration/hastexo"
CERTS_REPO: "https://{{ COMMON_GIT_MIRROR }}/hastexo/edx-certificates.git"
certs_version: "master"
forum_source_repo: "https://{{ COMMON_GIT_MIRROR }}/hastexo/edx-forum.git"
forum_version: "master"
xqueue_source_repo: "https://{{ COMMON_GIT_MIRROR }}/hastexo/edx-xqueue.git"
xqueue_version: "hastexo/integration/hastexo"
NOTIFIER_SOURCE_REPO: "https://{{ COMMON_GIT_MIRROR }}/hastexo/edx-notifier.git"
NOTIFIER_VERSION: "master"
```

At this point, make sure to create a Swift container for this deployment: it'll
be needed during the initial playbook run.  You can do so from your own
computer, as long as the `openstack` command line client is installed and the
appropriate OpenStack credentials are loaded into the environment.  By default,
the sample Ansible variables will configure edX to use the "lms.example.com"
container.  Run the following to create the container with read permissions for
connections originating from the site itself:

    Note: You can change the site name by editing
    `edx-configuration-secrets/group_vars/all` and replacing `EDXAPP_SITE_NAME`
    with the desired domain.

```
site=lms.example.com
openstack container create $site
openstack container set --property "read_acl=.r:$site" $site
```

Take the opportunity to set the `SWIFT_LOG_SYNC_*` variables in
`group_vars/all`, even if you're not actually enabling Swift log
synchronization.  These Swift authentication variables will also be used by
default for edX report storage.

    Note: If you intend to use the hastexo XBlock, refer to the "Multiple nodes
    with the hastexo XBlock" section below, before continuing.  Avoid actually
    running the playbook in the last step therein: this will be accomplished in
    in this section.

Now, generate a passphrase-less keypair, copy it over to the other nodes, and
add the nodes to the host list so one doesn't have to manually confirm them
later.  Note that this will only work if authentication forwarding is properly
configured, and if you're logged in as the default user, "ubuntu".

```sh
# Create a new key pair
export KEYFILE=~/.ssh/id_rsa
ssh-keygen -t rsa -N "" -f $KEYFILE

# Disable strict host key checking
echo "StrictHostKeyChecking no" > ~/.ssh/config

# Deploy it
ips=$(ansible -i ~/edx-configuration-secrets/inventory.py all --list-hosts)
for ip in $ips; do
    ssh-keyscan $ip >> ~/.ssh/known_hosts
    ssh-copy-id -i $KEYFILE $ip
done
```

At this point, it is best if you reconnect to the deploy node **without** agent
forwarding, as the latter can interfere with the Ansible deployment.  Ideally,
start a screen session as well, so a disconnection will not stop the Ansible
run:

```
exit
ssh ubuntu@<deploy_ip>
screen
```

Be sure to run the `inventory.py` dynamic inventory generator, as opposed to
the static `intentory.ini`, meant for single node deployments.  Also, set
`migrate_db=yes` on this first run, to ensure that the databases and tables are
properly created.

```
cd ~/edx-configuration/playbooks
ansible-playbook -i ../../edx-configuration-secrets/inventory.py openstack-multi-node.yml -e migrate_db=yes
```

This playbook run may take one hour or more.  After it's done, log out of the
deploy node and edit your local /etc/hosts.  If you didn't change any of the
example variables, enter the following, substituting `app_ip` for the IP
address of the app server pool you can obtain with the following command:

```
openstack stack output show $stack app_ip
vim /etc/hosts
---
<app_ip> lms.example.com studio.example.com
```

You should now be able to access both the LMS and Studio using the following
HTTPS URLs, respectively:

* https://lms.example.com
* https://studio.example.com

To deploy additional application servers within a previously deployed
stack, use the `openstack stack update` command to increase the `app_count`
stack parameter:

```
openstack stack update --existing --parameter app_count=<new_num> $stack
```

If you removed app servers, there's nothing else you need to do.

If you added app servers, however, log back into the deploy node, and run the
multi-node playbook again, limiting the run to the app servers and avoiding
database migrations:

```
cd /var/tmp/edx-configuration/playbooks
ansible-playbook -i ../../edx-configuration-secrets/inventory.py openstack-multi-node.yml --limit app_servers
```

#### Multiple nodes with the hastexo XBlock

If you want to deploy the hastexo XBlock together with Open edX to your multi
node cluster, go back to your deploy node and:

1. Locate the following variables in
   `/var/tmp/edx-configuration-secrets/group_vars/all` (which you created
   above) and change them as described.  Set the `os_*` variables to the
   OpenStack cloud of your choice.

```
EDXAPP_EXTRA_REQUIREMENTS:
  - name: "git+https://github.com/hastexo/hastexo-xblock.git@master#egg=hastexo-xblock"
EDXAPP_ADDL_INSTALLED_APPS:
  - 'hastexo'
EDXAPP_XBLOCK_SETTINGS:
  hastexo:
    providers:
      default:
        os_auth_url: ""
        os_auth_token: ""
        os_username: ""
        os_password: ""
        os_user_id: ""
        os_user_domain_id: ""
        os_user_domain_name: ""
        os_project_id: ""
        os_project_name: ""
        os_project_domain_id: ""
        os_project_domain_name: ""
        os_region_name: ""
```

2. Add the `gateone` role to `openstack-multi-node.yml` under the `app_servers`
   section (the last one).

```
$ cd ~/edx-configuration/playbooks
$ vim openstack-multi-node.yaml
...
- certs
- demo
- gateone
```

3. Run that playbook, limitting the run to the `app_servers`:

```
$ ansible-playbook -i ../../edx-configuration-secrets/inventory.py openstack-multi-node.yml --limit app_servers
```

#### Working with app server master images

If you have more than just a few app servers, maintaining them directly with
ansible is very inefficient.  To solve this, an `edx-app-master.yaml` template
is provided.  With it, you can create a golden master image of a pristine app
server, taking advantage of the multi-node stack you may already have.

To deploy a master app server to an existing multi-node stack, you can either
request it to be created by the the existing stack, or you you can use the
`edx-app-master.yaml` template directly.


##### Requesting a new app master

To request the existing stack to create a new app master server, invoke:

```
stack=<existing_stack_name>
master_base_image=<master_base_image>
openstack stack update \
    --existing \
    --parameter enable_app_master=1 \
    --parameter app_master_image=$master_base_image \
    $stack
```

Now, run the `openstack-multi-node.yaml` playbook on the app master from the
deploy node.  You may also want to run database migrations at this point:

```
cd ~/edx-configuration/playbooks
ansible-playbook \
    -i ../../edx-configuration-secrets/inventory.py \
    --limit app_master [-e migrate_db=yes] \
    openstack-multi-node.yml
```

After the run finishes, go back to your local terminal to stop the server, and
create an image from it:

```
stack=<your_stack_name>
image=<your_image_name>
eval $(openstack stack output show $stack app_master_id --format shell)
openstack server stop ${output_value}
openstack server image create --name $image ${output_value}
```

Run the following to keep tabs on the image creation.

```
openstack image show $image
```

Once it's active, you're free to disable the app master server and
simultaneously configure the app servers to use the new image:

```
openstack stack update \
    --existing \
    --parameter enable_app_master=0
    --parameter app_image=$image \
    $stack
```


##### Using `edx-app-master.yaml` directly

To use the `edx-app-master.yaml` template directly, you must set a few
mandatory parameters when invoking Heat:

- `name`, the name of the master server
- `image`, the base image to use
- `flavor`, the flavor to use
- `key_name`, the name of the Nova keypair to use
- `network`, the id of the existing stack's management network
- `security_group`, the id of the existing stack's server security group

You can find out the management network ID and security group ID by issuing:

```
openstack stack resource show <existing_stack_name> management_net | grep physical_resource_id
openstack stack resource show <existing_stack_name> server_security_group | grep physical_resource_id
```

Where `<existing_stack_name>` is the name of the previously created multi-node
stack.

Finally, this is how you would create the auxiliary stack:

```
openstack stack create \
    --template heat-templates/hot/edx-app-master.yaml \
    --parameter name=<server_name> \
    --parameter image=<app_master_image> \
    --parameter flavor=<server_flavor> \
    --parameter key_name=<key_name> \
    --parameter network=<existing_network> \
    --parameter security_group=<existing_security_group> \
    <stack_name>
```

To verify that the stack has reached the `CREATE_COMPLETE` state, run:

```
openstack stack show <stack_name>
```

Once stack creation is complete, you can use `openstack stack output show` to
retrieve the internal IP address of the app master server:

```
openstack stack output show <stack_name> server_ip
...
"192.168.122.206"
```

Now, SSH into the existing stack's deploy node:

```
openstack stack output show <existing_stack_name> deploy_ip
ssh ubuntu@<deploy_ip>
```

You'll create a static inventory file, `app_master.ini`, containing the
existing `backend_servers`, and only the app master under `app_master`:

```
vim /var/tmp/edx-configuration-secrets/app_master.ini
...
[app_master]
192.168.122.206

[backend_servers]
192.168.122.111
192.168.122.112
192.168.122.113
```

You can find out what are the existing backend servers by running:

```
/var/tmp/edx-configuration-secrets/openstack.py --list
```

Now, run the `openstack-multi-node.yaml` playbook using this inventory file on
the `app_master` group.  You may also want to run database migrations at this
point:

```
cd /var/tmp/edx-configuration/playbooks
ansible-playbook \
    -i ../../edx-configuration-secrets/app_master.ini \
    --limit app_master [-e migrate_db=yes] \
    openstack-multi-node.yml
```

After the run finishes, go back to your local terminal stop the server, and
create an image from it:

```
openstack server stop <server_name>
openstack server image create --name <image_name> <server_name>
```

Run the following to keep tabs on the image creation.

```
openstack image show <image_name>
```

Once it's active, you're free to delete the app master stack:

```
openstack stack delete <stack_name>
```

And finally, you can use the image to update your existing stack.  For example:

```
openstack stack update \
    --existing \
    --parameter app_image=<image_name> \
    <existing_stack_name>
```

## Firing up an analytics stack

Create the stack using the existing stack's management network and security
group:

```
openstack stack create \
    --template heat-templates/hot/edx-analytics-server.yaml \
    --parameter name=<server_name> \
    --parameter image=<image> \
    --parameter flavor=<flavor> \
    --parameter key_name=<key_name> \
    --parameter network=<existing_network> \
    --parameter security_group=<existing_security_group> \
    --parameter public_net_id=<uuid> \
    <analytics_stack_name>
```

The analytics server's default internal IP is `192.168.122.120`.  Deploy the
stack's SSH key pair to it from the deploy node:

```
ip=192.168.122.120
ssh-keyscan $ip >> ~/.ssh/known_hosts
ssh-copy-id -i ~/.ssh/id_rsa $ip
scp ~/.ssh/id_rsa* $ip:.ssh/
```

You'll create a static inventory file, `analytics.ini`, containing the
existing `backend_servers`, and only the analytics server under
`analytics_servers`:

```
vim /var/tmp/edx-configuration-secrets/analytics.ini
...
[analytics_servers]
192.168.122.120

[backend_servers]
192.168.122.111
192.168.122.112
192.168.122.113
```

You can find out what are the existing backend servers by running:

```
/var/tmp/edx-configuration-secrets/openstack.py --list
```

Now, run the `openstack-analytics.yaml` playbook using this inventory file on
the `analytics_servers` group.

```
cd /var/tmp/edx-configuration/playbooks
ansible-playbook \
    -i ../../edx-configuration-secrets/analytics.ini \
    -e migrate_db=yes \
    openstack-analytics.yml
```

SSH into the the analytics node for the following:

```
ssh 192.168.122.120
```

Set up the pipeline in a new virtual env:

```
# Create a new virtualenv for the pipeline, and activate it
virtualenv pipeline
. pipeline/bin/activate

# Clone the repository and bootstrap it
git clone https://github.com/edx/edx-analytics-pipeline
cd edx-analytics-pipeline
make bootstrap
```

Copy the sample `devstack.cfg` configuration file and change it as follows:

```
sudo cp ~/edx-analytics-pipeline/config/devstack.cfg /edx/etc/edx-analytics-pipeline/override.cfg
sudo vim /edx/etc/edx-analytics-pipeline/override.cfg
...
[elasticsearch]
host = http://192.168.122.111:9201/
```

Test it with a simple task that counts daily events.  This will run through the
installation procedure and may take a while.  On subsequent invocations,
however, it will be possible to skip it.

```
remote-task \
    --host localhost \
    --repo https://github.com/edx/edx-analytics-pipeline \
    --user ubuntu \
    --override-config /edx/etc/edx-analytics-pipeline/override.cfg \
    --remote-name analyticstack \
    --wait TotalEventsDailyTask \
    --interval 2016 \
    --output-root hdfs://localhost:9000/output/ \
    --local-scheduler
```

Now import enrollments, skipping the installation:

```
remote-task \
    --host localhost \
    --user ubuntu \
    --override-config /edx/etc/edx-analytics-pipeline/override.cfg \
    --remote-name analyticstack \
    --skip-setup \
    --wait CourseEnrollmentEventsTask \
    --interval 2016 \
    --local-scheduler
```

Finally, log in to Insights, making sure you're logged into the LMS with a
staff account.  If your OAUTH variables were set correctly when deploying up
the LMS, the `edx-multi-node.yaml` playbook should have already created the
correct Insights token on it.

If there's a 500 error on the /courses page, you're likely hitting a migration
bug where some tables weren't created properly.  To fix this, run the
following:

```bash
sudo -Hu insights bash
cd
. venvs/insights/bin/activate
. insights_env
cd edx_analytics_dashboard
./manage.py migrate --run-syncdb --settings=analytics_dashboard.settings.production
exit
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
that Ansible can connect to the backend node without human intervention.  (You
can skip this step if you've already done it as part of initial stack
deployment.)

```sh
# Create a new key pair
export KEYFILE=~/.ssh/id_rsa
ssh-keygen -t rsa -N "" -f $KEYFILE

# Disable strict host key checking
echo "StrictHostKeyChecking no" > ~/.ssh/config

# Deploy it
ips=$(~/edx-configuration/playbooks/openstack/inventory.py --list | grep 192 | tr -d ' ",')
for ip in $ips; do
    ssh-keyscan $ip >> ~/.ssh/known_hosts
    ssh-copy-id -i $KEYFILE $ip
done
```
