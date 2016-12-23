## Working with app server master images

If you have more than just a few app servers, maintaining them directly with
ansible is very inefficient.  To solve this, an `edx-app-master.yaml` template
is provided.  With it, you can create a golden master image of a pristine app
server, taking advantage of the multi-node stack you may already have.

To deploy a master app server to an existing multi-node stack, you can either
request it to be created by the the existing stack, or you you can use the
`edx-app-master.yaml` template directly.


### Requesting a new app master

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


### Using `edx-app-master.yaml` directly

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
