..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Support Router Advertisement Daemon (radvd) for IPv6
====================================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/neutron-ipv6-radvd-ra

Add support for radvd to Neutron, and use radvd as the preferred reference
implementation for IPv6 Router Advertisements and SLAAC. [RADVD]_ is
open-source software that implements advertisements of IPv6 router addresses
and IPv6 routing prefixes using NDP.


.. _problemdescription:

Problem description
===================

Router Advertisements (RAs) are fundamental to the IPv6 Neighbor Discovery
standard [RFC_4861]_. Without radvd, Neutron would need to rely on dnsmasq for
RA functionality, which has a number of shortcomings:

* For DHCP the dnsmasq process runs in its own (dhcp) namespace. For RAs we
  would need to run additional dnsmasq processes in router namespaces. Since we
  don't want to change the current DHCP functionality we would have to disable
  prefix delegation for the dnsmasq processes running in router
  namespaces. This would lead to a lot of overhead and scale issues.

* Unicast RAs are not supported by dnsmasq. Unicast RAs are a useful
  feature. For example, they can be used to provide internet access to specific
  end points when the default assignment is Unique Local Addresses (ULA).

* Some older versions of dnsmasq included with distributions have insufficient
  IPv6 support.

This proposal is intended to cover announcing RAs, providing the announcement
information requested from the ipv6_ra_mode attribute.


Proposed change
===============

If IPv6 is enabled, start a radvd process in the namespace of each
router. Enable the Neutron L3 agent to support radvd configuration and
updating.

The L3 agent will be updated to support radvd configuration based on the
attributes of the subnets associated with the router. Since radvd is configured
by a config file, the L3 agent will maintain a separate config file per router
namespace. The radvd process will need to be restarted when a configuration is
changed.

When a subnet is deleted, the associated radvd configuration needs to be
updated so that the prefix is no longer advertised. The L3 agent will modify
the config and restart the radvd process.

Some radvd settings will affect the configuration of DHCP. The DHCP agent will
be updated to treat subnet attributes and configure dnsmasq so that is is
compatible with how radvd is configured.

Reasonable default values for some radvd settings, like RA interval and
lifetime, will be chosen in the first implementation. Follow-up bugs or feature
requests can be filed to add Neutron config options for radvd settings.


Alternatives
------------

We could use dnsmasq for RA and SLAAC support. The shortcomings of this are
given in section :ref:`problemdescription`.


Data model impact
-----------------

None.


REST API impact
---------------

No change to the API, but the user/admin should know how to use the two IPv6
addressing attributes [TWO_MODE]_ to configure subnets for IPv6 addressing.

+-----------+-----------+-------+------------------------------------------------------------+
| ipv6      | ipv6      | radvd | Description                                                |
| ra        | address   | A,M,O |                                                            |
| mode      | mode      |       |                                                            |
+===========+===========+=======+============================================================+
| *N/S*     | *N/S*     | Off   | VM obtains IPv6 address based on non-OpenStack services on |
|           |           |       | the network, or only creates a link-local address (not     |
|           |           |       | sourced by OpenStack).                                     |
+-----------+-----------+-------+------------------------------------------------------------+
| *N/S*     | slaac     | Off   | VM obtains IPv6 address from external router using SLAAC   |
+-----------+-----------+-------+------------------------------------------------------------+
| *N/S*     | dhcpv6-   | Off   | VM obtains IPv6 address and optional info from dnsmasq     |
|           | stateful  |       | using DHCPv6 stateful                                      |
+-----------+-----------+-------+------------------------------------------------------------+
| *N/S*     | dhcpv6-   | Off   | VM obtains IPv6 address from external router using SLAAC   |
|           | stateless |       | and optional info from dnsmasq using DHCPv6 stateless      |
+-----------+-----------+-------+------------------------------------------------------------+
| slaac     | *N/S*     | 1,0,0 | VM obtains IPv6 address from radvd using SLAAC             |
+-----------+-----------+-------+------------------------------------------------------------+
| dhcpv6-   | *N/S*     | 0,1,1 | VM obtains IPv6 address and optional info from external    |
| stateful  |           |       | DHCPv6 server using DHCPv6 stateful                        |
+-----------+-----------+-------+------------------------------------------------------------+
| dhcpv6-   | *N/S*     | 1,0,1 | VM obtains IPv6 address from radvd using SLAAC and         |
| stateless |           |       | optional info from external DHCPv6 server using            |
|           |           |       | DHCPv6 stateless                                           |
+-----------+-----------+-------+------------------------------------------------------------+
| slaac     | slaac     | 1,0,0 | VM obtains IPv6 address from OpenStack radvd using SLAAC   |
+-----------+-----------+-------+------------------------------------------------------------+
| dhcpv6-   | dhcpv6-   | 0,1,1 | VM obtains IPv6 address from dnsmasq using DHCPv6 stateful |
| stateful  | stateful  |       | and optional info from dnsmasq using DHCPv6 stateful       |
+-----------+-----------+-------+------------------------------------------------------------+
| dhcpv6-   | dhcpv6-   | 1,0,1 | VM obtains IPv6 address from radvd using SLAAC and         |
| stateless | stateless |       | optional info from dnsmasq using DHCPv6 stateless          |
+-----------+-----------+-------+------------------------------------------------------------+
| slaac     | dhcpv6-   |       | *Invalid combination*                                      |
|           | stateful  |       |                                                            |
+-----------+-----------+-------+------------------------------------------------------------+
| slaac     | dhcpv6-   |       | *Invalid combination*                                      |
|           | stateless |       |                                                            |
+-----------+-----------+-------+------------------------------------------------------------+
| dhcpv6-   | slaac     |       | *Invalid combination*                                      |
| stateful  |           |       |                                                            |
+-----------+-----------+-------+------------------------------------------------------------+
| dhcpv6-   | dhcpv6-   |       | *Invalid combination*                                      |
| stateful  | stateless |       |                                                            |
+-----------+-----------+-------+------------------------------------------------------------+
| dhcpv6-   | slaac     |       | *Invalid combination*                                      |
| stateless |           |       |                                                            |
+-----------+-----------+-------+------------------------------------------------------------+
| dhcpv6-   | dhcpv6-   |       | *Invalid combination*                                      |
| stateless | stateful  |       |                                                            |
+-----------+-----------+-------+------------------------------------------------------------+

*N/S* = *Not Specified*


Security impact
---------------

There is no security impact directly related to radvd, but it should be noted
that the default security group settings for IPv6 interfaces allow RAs from Neutron
routers and/or from the subnet's default gateway.


Notifications impact
--------------------

None.


Other end user impact
---------------------

CLI changes?


Performance Impact
------------------

This change introduces one radvd instance per router namespace that is created
by neutron. The radvd instances may load new configuration data when interfaces
are added into or removed from the namespaces. Significant system performance
degradation is not expected.


Other deployer impact
---------------------

* install radvd
* config options


Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
   * <gessau> Henry Gessau

Other contributors:
   * <anthony-veiga> Anthony Veiga
   * <baoli> Robert Li
   * <leblancd> Dane Leblanc
   * <scollins> Sean Collins


Work Items
----------

* L3 agent
* dhcp agent


Dependencies
============

* IPv6 two attributes [BP_2ATTR]_.
* Update SG when gateway address changes [PV_SG_RA]_.
* Consolidated L3 agent? [BP_CONSL3]_
* radvd package [RADVD]_

Note: For each SLAAC-enabled subnet, the Neutron L3 Agent will need to either
detect or anticipate which specific IPv6 address is generated by each
host. Note that for the initial release of IPv6 in OpenStack, the ability to
anticipate or derive each SLAAC-generated address was added with the change set
for [BP_PV_SLAAC]_. This algorithm assumes that interface IDs used in forming
addresses are generated using EUI-64.

Testing
=======

Full unit test coverage.

Look into the feasibilty of adding functional tests. (Not currently considered
essential.)

Add tempest tests like:

* API tests for network/subnet create for each of the modes (see [API_TESTS]_):

   * unspec/unspec, SLAAC/SLAAC, stateful/stateful, stateless/stateless

* Scenario ping test for SLAAC/SLAAC, stateful/stateful, stateless/stateless


Documentation Impact
====================

How to configure IPv6 addressing. Overview for Neutron and specifically for
subnets.


References
==========

.. [RADVD] `Linux IPv6 Router Advertisement Daemon (radvd)
   <http://www.litech.org/radvd/>`_
.. [BP_2ATTR] `Two Attributes Proposal to Control IPv6 RA Announcement and
   Address Assignment
   <https://blueprints.launchpad.net/neutron/+spec/ipv6-two-attributes>`_
.. [RFC_4861] `Neighbor Discovery for IP version 6 (IPv6)
   <http://tools.ietf.org/html/rfc4861>`_
.. [TWO_MODE] `IPv6 Two Modes v3.0.pdf
   <https://www.dropbox.com/s/9bojvv9vywsz8sd/
   IPv6%20T wo%20Modes%20v3.0.pdf>`_
.. [BP_PV_SLAAC] `Provider Networking - upstream SLAAC support
   <https://blueprints.launchpad.net/neutron/+spec/
   ipv6-provider-nets-slaac>`_
.. [BP_CONSL3] `L3 agent consolidation
   <https://blueprints.launchpad.net/neutron/+spec/l3-agent-consolidation>`_
.. [API_TESTS] `Add IPv6 API test cases for Neutron Subnet API
   <https://review.openstack.org/100134>`_
.. [PV_SG_RA] `Router RA rule need to be updated when router is created after
   VM port
   <https://bugs.launchpad.net/neutron/+bug/1290252>`_
