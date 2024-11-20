============================================
Provider Networking - upstream SLAAC Support
============================================

It should be possible to create a Provider Network in Neutron, that uses an
upstream switch or router that advertises RA's - so that instances can use
SLAAC to configure their IPv6 networking, while still utilizing the Neutron
APIs for security group rules.

https://blueprints.launchpad.net/neutron/+spec/ipv6-provider-nets-slaac

Problem description
===================

A deployer of OpenStack that wishes to use IPv6 and already has
configured IPv6 on northbound networking devices, may wish to
advertise IPv6 routes by having non-OpenStack hardware transmit
ICMPv6 Router Advertisement packets.

.. image:: /images/juno/provider-network-slaac.png

Proposed change
===============

OpenStack Neutron should support this configuration, and display the
correct IP address for instances, since Neutron has enough information
to generate the EUI-64 address.

Alternatives
------------

None


Data model impact
-----------------

None

REST API impact
---------------

The Neutron REST API for IPv6 Subnets allows the following
combinations, as well as disallowing certain combinations. In short,
when **both** ipv6_ra_mode and ipv6_address_mode are set, they must be
equal. When only one attribute is set, the setting is automatically
valid.

+--------------------+--------------------+-------+---------------+
|   ipv6_ra_mode     | ipv6_address_mode  | valid |  RA Settings  |
+====================+====================+=======+===============+
| ATTR_NOT_SPECIFIED | ATTR_NOT_SPECIFIED | yes   | ANY           |
+--------------------+--------------------+-------+---------------+
| ATTR_NOT_SPECIFIED | slaac              | yes   | "A=1,M=0,O=0" |
+--------------------+--------------------+-------+---------------+
| ATTR_NOT_SPECIFIED | dhcpv6-stateful    | yes   | "A=0,M=1,O=1" |
+--------------------+--------------------+-------+---------------+
| ATTR_NOT_SPECIFIED | dhcpv6-stateless   | yes   | "A=1,M=0,O=1" |
+--------------------+--------------------+-------+---------------+
| slaac              | ATTR_NOT_SPECIFIED | yes   | "A=1,M=0,O=0" |
+--------------------+--------------------+-------+---------------+
| dhcpv6-stateful    | ATTR_NOT_SPECIFIED | yes   | "A=0,M=1,O=1" |
+--------------------+--------------------+-------+---------------+
| dhcpv6-stateless   | ATTR_NOT_SPECIFIED | yes   | "A=1,M=0,O=1" |
+--------------------+--------------------+-------+---------------+
| slaac              | slaac              | yes   | "A=1,M=0,O=0" |
+--------------------+--------------------+-------+---------------+
| dhcpv6-stateful    | dhcpv6-stateful    | yes   | "A=0,M=1,O=1" |
+--------------------+--------------------+-------+---------------+
| dhcpv6-stateless   | dhcpv6-stateless   | yes   | "A=1,M=0,O=1" |
+--------------------+--------------------+-------+---------------+
| slaac              | dhcpv6-stateful    | no    | UNDEF         |
+--------------------+--------------------+-------+---------------+
| slaac              | dhcpv6-stateless   | no    | UNDEF         |
+--------------------+--------------------+-------+---------------+
| dhcpv6-stateful    | slaac              | no    | UNDEF         |
+--------------------+--------------------+-------+---------------+
| dhcpv6-stateful    | dhcpv6-stateless   | no    | UNDEF         |
+--------------------+--------------------+-------+---------------+
| dhcpv6-stateless   | slaac              | no    | UNDEF         |
+--------------------+--------------------+-------+---------------+
| dhcpv6-stateless   | dhcpv6-stateful    | no    | UNDEF         |
+--------------------+--------------------+-------+---------------+

This blueprint will implement the functionality required to satisfy
IPv6 Subnets that are set with ipv6_address_mode set to 'slaac', and
ipv6_ra_mode not set.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

* This change will require support in python-neutronclient for the two
  IPv6 subnet attributes - which `is currently in review <https://review.openstack.org/#/c/75871/>`_

Performance Impact
------------------

None

Other deployer impact
---------------------

* This change will change the behavior of Neutron in specific
  configurations, when the IPv6 attributes for Subnets are set.
  Previously, the attributes were no-ops.

Developer impact
----------------

None


Implementation
==============

* Subnets will be created with the `ipv6_address_mode` set to `slaac`
  and `ipv6_ra_mode` **not** set.

* Neutron will calculate the IP address of a port, using the MAC address
  and the CIDR.

Assignee(s)
-----------

Primary assignee:
        scollins

Work Items
----------

* `Support Subnets that are configured by external RAs <https://review.openstack.org/#/c/86044/>`_

* `Ensure entries in dnsmasq belong to a subnet using DHCP <https://review.openstack.org/#/c/64578/>`_

Dependencies
============

* The IPv6 Subnet Attributes must be returned in API calls.

Testing
=======

* Add unit tests to support Subnets created with only the
  `ipv6_address_mode` set.

* Verify that ports with allocations from a subnet with
  `ipv6_address_mode` set are not touched by the DHCP agent.

Documentation Impact
====================

Documentation about this network configuration will need to be
written.


References
==========

* `Devstack for IPv6 in the Comcast lab <http://lists.openstack.org/pipermail/openstack-dev/2014-February/026589.html>`_
