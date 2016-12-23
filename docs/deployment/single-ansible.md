# Deploying a single-node environment

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

## Single node with the hastexo XBlock

If you want to deploy the hastexo XBlock together with Open edX to your single
node, go back to your installed node and:

- Locate the following variables in
  `/var/tmp/edx-configuration-secrets/host_vars/localhost` (which you
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

- Check out the `hastexo/integration/base` branch of
  edx-configuration, which contains the `gateone` role:

```
$ cd /var/tmp/edx-configuration
$ git checkout -b hastexo/integration/base origin/hastexo/integration/base
```

- Add the `gateone` role to `openstack-single-node.yml` and rerun that
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