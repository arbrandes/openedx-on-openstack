# Deploying a multi-node environment

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

## Multiple nodes with the hastexo XBlock

If you want to deploy the hastexo XBlock together with Open edX to your multi
node cluster, go back to your deploy node and:

- Locate the following variables in
  `/var/tmp/edx-configuration-secrets/group_vars/all` (which you
  created above) and change them as described.  Set the `os_*`
  variables to the OpenStack cloud of your choice.

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

- Add the `gateone` role to `openstack-multi-node.yml` under the
  `app_servers` section (the last one).

```
$ cd ~/edx-configuration/playbooks
$ vim openstack-multi-node.yaml
...
- certs
- demo
- gateone
```

- Run that playbook, limitting the run to the `app_servers`:

```
$ ansible-playbook -i ../../edx-configuration-secrets/inventory.py openstack-multi-node.yml --limit app_servers
```
