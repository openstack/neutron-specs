
..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Get me a Network
====================================

In some cloud deployments using Neutron, there is a need for each tenant
to configure resources, like a network, subnet and router, before they
are able to boot a VM. This can be a difficult proposition for those that
do not understanding networking, or do not want to understand it - they
just want a running VM. In this document we will describe ways for an
administrator to configure their cloud networking such that a tenant
does not have to do the heavy lifting typically required.


Problem Description
===================

How does a tenant boot a VM without creating a Neutron network themselves?

There are a number of options.

1. The cloud administrator can create a shared private network

A shared private network is a resource that all tenant's can see
and use, for example, it will be in the output of 'neutron net-list'.
If it is the only network visible to a tenant, then by default, a
'nova boot' will use it as the network for a VM.

An administrator can create this network with just a few commands,
for example:

::

    $ neutron net-create --shared private-shared-network
    $ neutron subnet-create --name private-shared-subnet private-shared-network 10.0.0.0/24
    $ neutron router-create private-shared-router
    $ neutron router-interface-add private-shared-router private-shared-subnet
    $ neutron router-gateway-set private-shared-router public-network

Additionally an administrator could add an IPv6 subnet to the network
as well, giving all VMs IPv6 addresses automatically

Pros:

Easy to configure, with no tenant interaction.

Cons:

Since this network is shared by multiple tenants, they also share the
same broadcast domain, so it could be seen as less secure since inter-VM
communication is possible. For this reason it is recommended that
security groups are enabled by default.

Another con is that this shared network could become 'full' if the
address range is too small, or the VM count becomes too large. A reasonable
large default subnet should be created for this reason.

2. The cloud administrator can create a provider network

Similar to a shared private network, a provider network can be a shared
resource available to all tenants. The difference is that it does not
require a Neutron router, that function is provided by the infrastructure.

An administrator can create this network with just a few commands,
for example:

::

    $ neutron net-create provider-shared-network --provider:network_type flat --provider:physical_network eth0
    $ neutron subnet-create --name private-shared-subnet private-shared-network 10.0.0.0/24

Pros:

Easy to configure, with no tenant interaction.

Cons:

Similar to a shared private network.

3. The cloud administrator can provision a Neutron network for each tenant

As part of tenant creation, for example, after Keystone has been called
to create the tenant, an administrator could automatically configure
Neutron resources - network, subnet and router. This could be part of a
larger post-creation step specific to the deployment.

Pros:

Every tenant gets a private network for their VMs.

Cons:

Increased resource usage, since each tenant will have their own router
and DHCP server configured.

NOTE: This method will not be implemented as part of this specification.

4. Change Neutron to auto-allocate resources

Neutron could be changed to automatically allocate a network, subnet
and router when Nova calls it when booting a VM. This code does not
exist today, but given this was the primary reason for this spec,
will be proposed as part of devref documentation when landing this
feature.

Pros:

Every tenant gets a private network for their VMs.

Cons:

More complicated. Still requires administrator to define default
values or resources to be used to allocate the network. Possible
race conditions if many simultaneous calls are made.


Proposed Change
===============

Document how a deployer can create a shared private network and connect
it to their physical network. Most of this is in the example above.
We will also document how a provider network can be created. This will
cover options 1 and 2 above.

For option 4, detailed implementation will be done with devref
documentation in-tree as part of the change.


Future Work
---------------

A future enhancement could be to update the Neutron network model
with a new field to identify these "default" shared networks. This would
allow the Nova or Neutron code to filter them out if there is more than one
network, avoiding the dreaded:

ERROR (BadRequest): Multiple possible networks found, use a Network ID to be
more specific. (HTTP 400) (Request-ID: req-b429d44c-d6e8-4e29-9425-928c2dc22411

that can happen when more than one network is available at boot time.

This could be implemented as a new flag, or as a new preference value
that could be used to filter networks by priority. This would have API and
CLI impact, and affect both the Nova and Neutron code. This work would
need to specified in another document.

Gerrit Topic
------------

To enable reviewers to find offered patches implimenting the agreed upon
work the gerrit topic `get-me-a-network`_ will be used.

.. _get-me-a-network: https://review.openstack.org/#/q/topic:get-me-a-network+-status:abandoned,n,z

References
==========

There is a long history of trying to do orchestration-type magic inside
Neutron and the results have made it difficult for external orchestration
tools[1], added race conditions and caused disputed ownership of resources[2].
We want to avoid the same kinds of issues with this work. For this reason, we
need to be able to distinguish between instances that are in a VPC (i.e. have
advanced networking) and those that are not.  For example, AWS does this today.

[1] http://lists.openstack.org/pipermail/openstack-dev/2014-April/032098.html
[2] https://bugs.launchpad.net/nova/+bug/1158684

Etherpad from the Liberty summit:

https://etherpad.openstack.org/p/YVR-neutron-get-me-a-network
