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



