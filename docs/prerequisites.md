# Prerequisites

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

```bash
openstack network list
openstack network show <networkname>
```

- Create a Nova keypair for yourself (`openstack keypair create`).
- Upload an Ubuntu 14.04 image for your tenant into Glance:

```bash
wget https://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --file trusty-server-cloudimg-amd64-disk1.img \
  --name ubuntu-14.04-server-cloudimg
```

Alternatively, you may ask your cloud administrator do the same for
 you, and mark the image public by adding the `--public` option.
