# Firing up an analytics stack

Create the stack using the existing stack's management network and security
group:

```bash
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

```bash
ip=192.168.122.120
ssh-keyscan $ip >> ~/.ssh/known_hosts
ssh-copy-id -i ~/.ssh/id_rsa $ip
scp ~/.ssh/id_rsa* $ip:.ssh/
```

You'll create a static inventory file, `analytics.ini`, containing the
existing `backend_servers`, and only the analytics server under
`analytics_servers`:

```bash
vim /var/tmp/edx-configuration-secrets/analytics.ini
```

```ini
[analytics_servers]
192.168.122.120

[backend_servers]
192.168.122.111
192.168.122.112
192.168.122.113
```

You can find out what are the existing backend servers by running:

```bash
/var/tmp/edx-configuration-secrets/openstack.py --list
```

Now, run the `openstack-analytics.yaml` playbook using this inventory file on
the `analytics_servers` group.

```bash
cd /var/tmp/edx-configuration/playbooks
ansible-playbook \
    -i ../../edx-configuration-secrets/analytics.ini \
    -e migrate_db=yes \
    openstack-analytics.yml
```

SSH into the the analytics node for the following:

```bash
ssh 192.168.122.120
```

Set up the pipeline in a new virtual env:

```bash
# Create a new virtualenv for the pipeline, and activate it
virtualenv pipeline
. pipeline/bin/activate

# Clone the repository and bootstrap it
git clone https://github.com/edx/edx-analytics-pipeline
cd edx-analytics-pipeline
make bootstrap
```

Copy the sample `devstack.cfg` configuration file and change it as follows:

```bash
sudo cp ~/edx-analytics-pipeline/config/devstack.cfg /edx/etc/edx-analytics-pipeline/override.cfg
sudo vim /edx/etc/edx-analytics-pipeline/override.cfg
```

```ini
[elasticsearch]
host = http://192.168.122.111:9201/
```

Test it with a simple task that counts daily events.  This will run through the
installation procedure and may take a while.  On subsequent invocations,
however, it will be possible to skip it.

```bash
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

```bash
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
