..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Floating IPs for Routed Networks
================================

https://bugs.launchpad.net/neutron/+bug/1667329

This specification proposes the implementation of floating IPs for Routed
Networks.

Problem Description
===================

`Neutron Routed Networks`__ are composed of individual L2 segments stitched
together by L3 routers in the operator infrastructure. The operator has the
option to route the fixed IP addresses of the ports in such a network to the
external world, but this might not be feasible for IPv4 without a large enough
externally routable address range.

__ routed-networks_

Routed Networks were implemented in the first place to support a large number
of ports / VMs, beyond the constraints of networks based on a single L2
broadcast domain. Operators want to be able to support a number of ports / VMs
as large as possible and route a subset of them externally, without being
limited by the availability IPv4 address ranges.

Proposed Change
===============

This specification proposes to enable the association of ports in a Routed
Network with floating IP addresses (floating IPs). This will enable operators
and their users to create ports in routed networks with fixed IP addresses in
private ranges and assign to them scarce publicly routable IPv4 addresses only
when needed, through the association of floating IPs. This functionality will
require the changes described in the following sections.

Creation of subnets spanning segments
-------------------------------------

To support the use of floating IP's on a routed provider network, there
needs to be a way to create a subnet that is not bound to a segment. In its
current state, a subnet cannot be created on a network without specifying a
segment when one or more subnets on the network are already bound to a
segment.

A new subnet service type of ``network:routed`` will be introduced. When a
subnet supports this service_type it can be associated to the network of a
routed provider network rather than having to be associated to one of the
segments that comprises the routed network. This provides a way of expressing
the notion of a subnet that spans the segments of a routed network. As such,
this not only supports the use of floating IP's on routed networks but also
allows various neutron backends to handle these subnets in innovative ways.

The current definition of a routed provider network is a network in which
each subnet is associated to a different segment. This definition will change
subtly and include networks where 1 or more subnets with ``network:routed`` in
its ``service_types`` is associated with the network but none of the individual
segments comprising it.

To enable this, minor changes to ``_validate_segment()`` in
``neutron/db/ipam_backend_mixin.py`` that allow for subnets with this special
service type to exist outside of any segment.


Service subnets and floating IPs for routed networks
----------------------------------------------------

`Service subnets`__ enable operators to define valid port types for each subnet
on a network, without limiting networks to one subnet or manually creating
ports with a specific subnet ID. Using this feature, operators can ensure that
ports for instances and router interfaces, for example, always use different
subnets.

__ service-subnets_

To allow floating IP's to be created on a routed network, an operator will
leverage this functionality when creating the floating IP subnet. To create a
floating IP subnet that can be used on a routed provider network, the subnet
should be created with two ``service_types``: ``network:routed`` and
``network:floatingip``. ``network:routed`` allows the subnet to be pinned to
the network rather than a particular segment, and ``network:floatingip``
allows for floating IP's be allocated from the subnet.

No changes are anticipated to enable this to work properly. Documentation
will be updated to explain the workflow for operators. No change to the
current workflow for creating/associating/disassociating/deleting floating
IP's will be introduced.

DVR and routed provider networks
--------------------------------

No changes to the current operation of DVR need to be made in support of
these changes. Changes to documentation will be made to reflect the fact that
subnets on segments where you wish to have floating IP gateway ports built
should be created with the ```service_type`` of
``network:floatingip_agent_gateway`` should be used to indicate that compute
nodes can reach the routed external network on a given segment. For the
network node connectivity, a ``service_type`` of ``network:router_gateway``
should be used to indicate centralized router ports will be instantiated on
the proper segment(s).

Because only the ToR device providing connectivity to the network segment
would know how to reach the floating IP (DVR will send a gratuitous ARP),
connectivity from outside the segment must be provided host routes that steer
traffic to the appropriate segment. This can be achieved by enabling
neutron-dynamic-routing, enabling it to announce FIP reachability to
ToR routers, and ensuring ToR routers propogate the FIP host route with an
appropriate next-hop. Documentation will be added to explain how this can be
set up by an operator.


Floating IPs advertisement with BGP Dynamic Routing
---------------------------------------------------

`BGP Dynamic Routing`__ is a Neutron Stadium project that enables advertisement
of self-service (private) network prefixes to physical routers that support
BGP, thus removing the conventional dependency on static routes. A ``BGP
speaker`` advertises the self-service network prefixes to associated ``BGP
peers``, which represent the BGP capable physical routers in the operator
infrastructure.

__ bgp-dynamic-routing_

In its current state, neutron-dynamic-routing determines next-hops for floating
IP's by identifying the relevant router and floating IP agent gateway ports
and mapping floating IP's to their proper endpoint. In this way, it is
completely agnostic of segments and is capable of properly discovering and
announcing the correct next-hop for a /32 host route of a floating IP.

Security Impact
---------------

This change is not expected to pose any new security concerns.

Other End User Impact
---------------------

No client impacts are anticipated. The network guide will be updated with
proper documentation and example configurations.

Future Work
-----------

For some operators, there is a desire to allow for floating IP's to be
associated to a port without first connecting a router to an external network
and a tenant network. This can be implemented in many ways. One way is by
relying on a cloud-init-like process to set the FIP as a loopback address
inside a VM and rely on a BGP process to announce the FIP with the fixed IP
as the next-hop address for the floating IP. With the implementation described
in previous sections in place, a similar scheme would be the next logical
iteration of routed provider networks.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* `Ryan Tidwell <https://launchpad.net/~ryan-tidwell>`_

Other contributors:

* `Miguel Lavalle <https://launchpad.net/~minsel>`_

Work Items
----------

#. Introduce the new ``network:routed`` subnet service type.
#. Refactor network segment integrity checks to allow subnets with a
   ``service_type`` of ``network:routed`` to be associated directly to a
   network.

Dependencies
============

None

Testing
=======

Tempest Tests
-------------

There are currently no tempest tests for routed provider networks. However,
some tests are being developed and tests that cover floating IP creation on
routed provider networks can eventually be added to these tests.

Functional Tests
----------------

Unknown

API Tests
---------

The following tests can be added to the suite of scenario tests being developed
in https://review.opendev.org/#/c/665155/ :

- Floating IP subnets can be created on a routed provider network
- Floating IP's can be created on a routed provider network
- Floating IP's from the same subnet can be associated to ports across
  different segments
- Tests in neutron-dynamic-routing to assert proper route discovery on
  routed provider networks


Documentation Impact
====================

Yes

User Documentation
------------------

- Document new behavior for creating floating IP subnets
- Document how to use in conjunction with neutron-dynamic-routing
- Example configurations of routed networks, ToR router BGP config, and
  neutron-dynamic-routing.

Developer Documentation
-----------------------

A new section to the Neutron devref will be added describing the implementation
of floating IPs for routed networks.

References
==========

.. _routed-networks: https://specs.openstack.org/openstack/neutron-specs/specs/newton/routed-networks.html
.. _bgp-dynamic-routing: https://docs.openstack.org/ocata/networking-guide/config-bgp-dynamic-routing.html
.. _service-subnets: https://docs.openstack.org/neutron/latest/admin/config-service-subnets.html
.. _DVR: https://docs.openstack.org/neutron/latest/admin/deploy-ovs-ha-dvr.html
