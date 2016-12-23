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
