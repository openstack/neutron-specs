=====================================================
Support DHCPv6 stateless and stateful mode in Dnsmasq
=====================================================

As [Spec_RADVD]_ is proposed to use radvd as the preferred reference
implementation for IPv6 Router Advertisements and SLAAC, this spec is
to allow tenant VM to obtain stateful dhcpv6 address or stateless
dhcpv6 address by Dnsmasq when ipv6_address_mode of a tenant subnet is set.

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/dnsmasq-ipv6-dhcpv6-stateless
https://blueprints.launchpad.net/neutron/+spec/dnsmasq-ipv6-dhcpv6-stateful

Problem description
===================

1. When dhcpv6 stateful is set as IPv6 address mode of a tenant subnet,
OpenStack admin wishes tenant VMs can obtain IPv6 address and optional
info (such as DNS info) from OpenStack network service.

2. When dhcpv6 stateless is set as IPv6 address mode of a tenant subnet,
router advertisement is taken care of by either external router or
OpenStack managed RADVD. OpenStack admin wishes tenant VMs can obtain
IPv6 address by SLAAC and obtain optional info from OpenStack network
service.

::

   +----------------------------------------------------------------------+
   |                                                                      |
   |                                                                      |
   |                                                                      |    +---------------------------------+
   |  +------------------------------+     +---------------------------+  |    |                                 |
   |  |dhcp namespace                |     | router                    |  |    |       +--------------+          |
   |  |  +-----------+               |     | namespace  +----------+   |  |    |       |              |          |
   |  |  |  dnsmasq  |               |     |            |   RADVD  |   |  |    |       |              |          |
   |  |  +-+---+-----+--------+      |     |       +----+--------+++   |  |    |  +------->           |          |
   |  |    |   |   qdhcp-xxxx |      |     |       |    qr-xxxx  ||    |  |    |  |    |              |          |
   |  |    |   +-------+------+      |     |       +----+---------+    |  |    |  | +----->  VM       |          |
   |  |    |           |             |     |            |        |     |  |    |  | |  |              |          |
   |  +------------------------------+     +---------------------------+  |    |  | |  +-------+------+          |
   |       |           |                                |        |        |    |  | |          |                 |
   |       |        +--+--------------------------------+-+      |        |    |  | |          |                 |
   |       |        |              br-int                 |      |        |    |  | | +--------+--------+        |
   |       |        +------------------+------------------+      |        |    |  | | |    br-int       |        |
   |       |                           |                         |        |    |  | | +--------+--------+        |
   |       |                           |                         |        |    |  | |          |                 |
   |       |                    +------+------+                  |        |    |  | |          |                 |
   |       |                    |   br-eth0   |                  |        |    |  | |  +---------------+         |
   |       |                    +------+------+                  |        |    |  | |  |     br-eth0   |         |
   |       |                           |                         |        |    |  | |  +---------------+         |
   |       |                           |                         |        |    |  | |          |                 |
   |       |                     +-----+-----+                   |        |    |  | |   +------+-------+         |
   |       |                     |   eth0    |                   |        |    |  | |   |     eth0     |         |
   +-----------------------------+-----+-----+----------------------------+    +--------+-------+------+---------+
           |                           |                         |                | |           |
           |                           |                         |                | |           |
           |                           +--------------------------------------------------------+
           |                                                     |                | |
           +------------------------------------------------------------------------+
                                IPv6 address/optional info       |    RA          |
                                                                 +----------------+

Proposed change
===============

According to [DNSMASQ_MANPAGE]_:

1. DHCPv6 stateless mode: dnsmasq in ‘static’ mode with ‘–dhcp-optsfile’
option specified can be leveraged to use dnsmasq as a simple stateless
dhcp server.

2. DHCPv6 stateful mode: dnsmasq in ‘static’ mode with ‘–dhcp-hostsfile’
and ‘–dhcp-optsfile’ options specified can be leveraged to use dnsmasq
as stateful dhcp server.

Alternatives
------------

None


Data model impact
-----------------

None

REST API impact
---------------

In Icehouse release, ipv6_ra_mode and ipv6_address_mode are introduced
to subnet API to enable tenant network IPv6 support.

Rest API change for IPv6 modes in Icehouse reference: [IPv6_MODES_REST]_

This blueprint will implement the functionality required to satisfy
IPv6 Subnets with ipv6_address_mode set to ‘dhcpv6-stateful’ or
‘dhcpv6-stateless’.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

* This change will require support in python-neutronclient for the two
  IPv6 subnet attributes - which is currently in `review <https://review.openstack.org/#/c/75871/>`_

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

* Subnets will be created with ‘ipv6_address_mode’ set to
  ‘dhcpv6-stateful’ or ‘dhcpv6-stateless’.

* If no dnsmasq process for subnet’s network is launched,
  Neutron will launch new dnsmasq process on subnet’s dhcp port
  in ‘qdhcp-‘ namespace. If previous dnsmasq process is already
  launched, restart dnsmasq with new configuration.

* Neutron will update dnsmasq process and restart it when subnet gets
  updated.

Assignee(s)
-----------

Primary assignee:
        * Shixiong
        * xuhanp

Work Items
----------

* Break down this `code review <https://review.openstack.org/#/c/70649>`_
  into smaller patches, and submit new code review for dhcpv6
  stateful/stateless mode.

Dependencies
============

* The assumption of this spec is Router Advertisement is provided by
  either provider network router or OpenStack Network Service (One
  implementation is RADVD [Spec_RADVD]_).

* Bump DNSMASQ version to 2.63 to support IPv6 tag. [DNSMASQ_VERSTION]_

Testing
=======

* Add unit tests to support Subnets created with the ipv6_address_mode
  set to ‘dhcpv6-stateful’ or ‘dhcpv6-stateless.

* Verify that dnsmasq can be launched with expected mode and options,
  in correct namespace.

* Add tempest API and scenairo tests for stateful and stateless dhcpv6.
  Reference: [API_TESTS_IPV6]_

Documentation Impact
====================

Documentation about this network configuration will need to be
written.


References
==========

.. [Spec_RADVD] `Spec for adding support for radvd for IPv6 SLAAC
   <https://review.openstack.org/#/c/101306>`_

.. [DNSMASQ_MANPAGE] `DNSMASQ man page
   <http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html>`_

.. [API_TESTS_IPV6] `Add IPv6 API test cases for Neutron Subnet API 254
   <https://review.openstack.org/100134>`_

.. [DNSMASQ_VERSTION] `DNSMASQ minimum version for IPv6
   <https://bugs.launchpad.net/neutron/+bug/1233339>`_

.. [IPv6_MODES_REST] `Rest API change for IPv6 modes in Icehouse`
   <http://docs-draft.openstack.org/43/88043/9/gate/gate-neutron-specs-docs/82c251a/doc/build/html/specs/juno/ipv6-provider-nets-slaac.html#rest-api-impact>_
