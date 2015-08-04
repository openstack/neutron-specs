..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========
IPv6 Router
===========

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/ipv6-router

IPv6 and IPv4 have significant differences, not only on the IP address format,
but also on the protocol operations. In this specification, we will discuss how
to support IPv6 neutron router in terms of public access.

Problem Description
===================

For IPv4, a tenant can assign a private prefix to its network, and use
floatingip to gain access to the public network that is assigned with public IP
addresses. Floatingip is currently not supported for IPv6 in neutron. There is
a neutron spec for IPv6 floatingip that may be possible for later openstack
releases [IPV6_FIP]_. To gain public access with a neutron router that is
connected to a public network, a tenant can use a GUA address in its own
network.

For IPv4, public IP addresses need to be assigned to the gateway port so that
NAT traffic can terminate. For IPv6 with GUA tenant network, it's not necessary
to assign a GUA prefix (or any prefix) to the gateway port. This is because, by
default, an IPv6 network is always assigned with the LLA prefix fe80::/10
[IPV6_ADDR]_. And the same is true for the gateway port in a neutron router
with IPv6 enabled.

Note that the external network can still be associated with an explicit IPv6
subnet. Its use case will be explained in [IPV6_FIP]_.

It's possible that the upstream router runs RA. The RA may simply advertise a
default route. In the same time, it may also carry a prefix for SLAAC. In
either case, the neutron router should have a default route installed after the
RA message is processed. To cover the case where the upstream router doesn't
advertise a default route with RA, there needs to be a way to configure the
default nexthop in the neutron router.

Proposed Change
===============

The implementation of *router-gateway-set* API will be changed so that an IPv6
subnet is not required in the external network identified by the
*external-network-id*. To create the gateway port for a neutron router, the
internal implementation of the *port-create* API is invoked. Currently in the
IPv6 only case, without explicitly providing an IPv6 subnet, a Bad Request
exception will be generated. Assuming the gateway port is associated with the
fe80::/10 prefix, change will be made to ensure the gateway port is created
successfully.

When an IPv4 subnet is added in a router, check to make sure that the router's
gateway port is on an IPv4 public network.

To support explicit nexthop configuration of neutron router in the absence of
upstream RA, add an *ipv6-gateway* parameter in the l3 agent configuration file
as in below:

::

    ipv6-gateway = GATEWAY_LLA

The *GATEWAY_LLA* should be a valid LLA address. And if it's not a valid LLA
address, L3 agent will log a warning message. With a valid *ipv6-gateway* LLA
address, the l3 agent will install a default route with the nexthop being the
*GATEWAY_LLA*, and the outgoing interface the gateway port.

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

IPv6 Impact
-----------

This specification specifies the changes to support IPv6 router.

Other Deployer Impact
---------------------

This spec introduces a new l3 agent configuration parameter as described in above.

Developer Impact
----------------

If a plugin supports neutron router and IPv6, then it may need to be changed to
support the router API semantics change.

Also devstack change may be needed to support the IPv4 only, IPv6 only and dual
stack configuration.

Community Impact
----------------

This change is aligned with the community effort in support of IPv6 in neutron

Alternatives
------------

In the IPv6 only case, it's possible to create a fake ipv6 subnet and use
it to create the gateway port.

In an IPv6 only router, the neutron public network doesn't actually contain any
useful information currently. But removing it would require API change. In
addition, it may be useful in the future to partition the public network into
smaller broadcast domains.

Another idea is to have a flag associated with a neutron router to indicate if
the router is running in one of the three modes: IPv4 only, IPv6 only or dual
stack. This can make sure that the gateway port is created properly with
required subnets.

Is it necessary to provide a *disable_ipv6* option for a neutron network,
and/or an overall config option *disable_ipv6* in the neutron config file?

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  baoli

Other contributors:
  will add as needed

Work Items
----------

* Change due to router API semantics change
* create the gateway port in the absence of subnets
* Develop test cases
* Tempest tests

Dependencies
============

It may have a dependency on the l3 agent refactoring if it's found that change
is needed in the l3 agent.

Testing
=======

Tempest Tests
-------------

Tempest tests should be developed to ensure the neutron routers work
properly in IPv4 only, IPv6 only and dual stack environment.

Functional Tests
----------------

Functional tests are needed to ensure the gateway port is properly created in
IPv4 only, IPv6 only and dual stack cases.

API Tests
---------

Existing unit tests for the router APIs may need to be updated to
reflect the change. New unit tests may be needed to test the API semantic
changes.

Documentation Impact
====================

User Documentation
------------------

User guide to neutron router needs to be updated

Developer Documentation
-----------------------

API semantic changes need to be documented.

References
==========

.. [REFAC_L3]  `Kilo refactoring and restructuring the l3 agent <https://review.openstack.org/#/c/131535/>`_
.. [IPV6_ADDR] `IP Version 6 Addressing Architecture <https://tools.ietf.org/html/rfc4291>`_
.. [IPV6_FIP]  `IPv6 Floating IP Support <https://review.openstack.org/#/c/139731/>`_
