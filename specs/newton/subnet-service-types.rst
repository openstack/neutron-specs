..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Subnet Service Types
====================

RFE:
https://bugs.launchpad.net/neutron/+bug/1544768

Launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/service-subnets

The current IP allocation method for ports on the external network
utilizes the same strategy done for internal networks - choose
an IP from the subnet(s) associated with the network.  In this
scenario, all subnets are created equal, and there is no way
to give precedence to one over the other.

Since these IP addresses are typically Internet-routable, or at
least routable within the datacenter they are used in, they
should be considered scarce, and be assigned sparingly.

This blueprint aims to change the allocation strategy to better
utilize this scarce resource, and only use it for ports that
must have Internet-routable IP addresses, using secondary subnets
for IP allocations for other ports.


Problem Description
===================

In Neutron, each router with an interface on the external network consumes
one (or more) IP addresses from the subnet configured on it.  For example,
on the Network Node, an IP address is assigned to a project's (aka tenant's)
router in order to allow instances to access the external network
using this single SNAT IP address.

In the case of a router on a distributed (compute) node, which is also
connected to the external network, a single Floating IP (FIP) namespace
is created in order for each instance with a floating IP address to be
able to directly reach the external network.  The floating IP namespace
also consumes an IP address from the external subnet to use for all tenant
routers residing on this node, even though it is never used for north-south
routing of traffic - it is mainly used for reachability testing (aka - can
I reach that distributed router).

In this second case, there is no reason an IP address from a different,
secondary subnet, could not be used for the primary IP address inside
the FIP namespace, such that reachability can still be confirmed, but
public IP consumption is not impacted.

Also, when a router is just forwarding packets onto a transit
network for further processing by another device, like a carrier-grade
NAT, no NAT needs to be performed in the router, the packet just
follows the standard forwarding data path.  So using a subnet that
exists only within the datacenter is possible.

Being able to choose the subnet used for IP allocation in all these
cases is highly-desired by operators.


Proposed Change
===============

This spec proposes adding a new logical item, called 'service_type', to
the Subnet class object in Neutron, a list attribute that can specify
the usage type for the subnet.  A type must be one of the currently
valid values for the 'device_owner' field in the Port model
(port['device_owner']), for example:

    - 'network:floatingip_agent_gateway'
    - 'network:router_gateway'
    - ...

Since it is a list, it can contain more than one value in order to
specify IP addresses for multiple device_owner types that should be
allocated from the same subnet.

This list will be instantiated as a new table containing rows of
(subnet_id, service_type) values, such that a join from Subnets.id
can be made to determine the service_types associated with the
subnet.

When an IP allocation is performed, for example, when an external gateway
is added, and the plugin supports the 'service-type' extension, the IPAM
driver will return an address from the Subnet that has a 'service-type'
matching the port's device_owner field.

In the case where no Subnets with the given 'service-type' exist for
the port's device_owner, or the IP address pool of all matching subnets
is exhausted, the fallback is to use a Subnet without any type for IP
allocation.  This preserves backwards-compatibility with existing
deployments.

When all Subnets are created with a 'service-type', there is no
fallback strategy since a Subnet without a type does not exist.
This can be used to enforce strict IP allocation from only Subnets
where the device_owner matches.

When multiple Subnets exist with the given 'service-type', or there are
both a matching and one without a type, then they will be tried in
order, with the matching 'service-type' Subnet(s) being tried first.
For example, with a Floating IP agent gateway port, the precedence
order would be ['network:floatingip_agent_gateway', None].

If a Subnet is specified explicitly in the port create or update request, the
logic to select candidate Subnets described above will be skipped.  The
specified Subnet will be the only candidate regardless of its 'service-type'.

With the data model proposed, all of this can be accomplished by simply
ordering all the matching subnets correctly, putting 'service-type'
subnets first, followed by non-'service-type' subnets, in a single query.

Data Model Impact
-----------------

#. Service-type subnet extension is added
#. New table is created linking Subnet IDs with Service Types.  It will
   have the columns (subnet_id, service_type), where the subnet_id has
   a foreign key constraint on Subnets.id.
#. The primary key will be (subnet_id, service_type) to provide an index
   and uniqueness.

REST API Impact
---------------

- 'service_type' will be exposed to users via the extension API.
- Ability to create or update subnets with a service-type will be
   limited to the admin user, since it will only be affecting IP
   allocation on external networks.  Validation will be done to
   restrict the values to a known set of values supported by the
   extension.
- List Subnets of a given service-type.

Sample REST calls:

.. code::

  POST /v2.0/subnets
  {
        'subnet': {
            'network_id': '<NETWORK-ID>',
            'ip_version': 4,
            'cidr': '10.0.3.0/24',
            'allocation_pools': [{'start': '10.0.3.20', 'end': '10.0.3.150'}],
            'service_type': ['network:floatingip_agent_gateway']
        }
  }

  PUT /v2.0/subnets/SUBNET-ID/service_types/
  {
      'service_type': ['network:floatingip_agent_gateway']
  }

  GET /v2.0/subnets?service_type='network:floatingip_agent_gateway'

Command Line Client Impact
--------------------------

A new option will be added to the openstackclient, for example:

$ openstack subnet create {ARGS} --service-type SERVICE-TYPE SUBNET-NAME

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

Horizon will need to at least expose the service_type attribute on the
subnet.

Performance Impact
------------------

There should be almost no performance impact as this should just be
an additional table join in the IPAM code.

IPv6 Impact
-----------

This change will consider IPv4 and IPv6 equally.

Other Deployer Impact
---------------------

Deployers will not be required to do anything when upgrading code until
they want to make use of the new features provided by this spec.

New or existing deployments can make use of this new feature after
upgrading their code by adding a new Neutron subnet to the existing
external network, or change the 'service-type' of an existing subnet on
that network if required.

They might need to change configuration in their top-of-rack switches,
or whatever else needs to be done to have a secondary subnet on their
existing physical network.  That work is outside the scope of this spec.

Developer Impact
----------------

This change will have an effect on developers.  This blueprint contains API
and model changes that plugin providers will need to look at implementing.

Community Impact
----------------

Yes.  This change has been discussed on the ML, in Neutron meetings
(especially L3), at mid-cycles, and at the design summit.

Alternatives
------------

An alternative is to not use Public IP addresses as Floating IP addresses,
but instead use IP addresses that are routable within the datacenter
infrastructure.  Access to the public Internet is then accomplished via
a carrier-grade NAT device that sits between the two networks.


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* `Brian Haley <https://launchpad.net/~brian-haley>`_

Other contributors:

* `Carl Baldwin <https://launchpad.net/~carl-baldwin>`_

Please add your name here and attend the `L3 Subteam Meeting
<https://wiki.openstack.org/wiki/Meetings/Neutron-L3-Subteam>`_ if you'd
like to contribute.  If your name is here and you don't have any time to
contribute, let me know.  This section is really just my sticky note of
contributors who can play a role in this effort.

Work Items
----------

#. Subnet extension

   - For ML2, use the multi-provider extension data model.
   - Add new 'Subnet_service_types' table
   - Add logical service_type value to Subnet
   - API tests

#. Client support

#. IPAM support

   - When choosing an IP address to allocate to a port, IPAM needs
     to take into account the device_owner when selecting the subnet.


Dependencies
============

None.


Testing
=======

Tempest Tests
-------------

Unknown

Functional Tests
----------------

Unknown

API Tests
---------

- Subnet service-type resource (CRUD)


Documentation Impact
====================

Yes

User Documentation
------------------

Document the usage of the value in the Subnet

Developer Documentation
-----------------------

Yes

References
==========

.. _rfe: https://bugs.launchpad.net/neutron/+bug/1544768
