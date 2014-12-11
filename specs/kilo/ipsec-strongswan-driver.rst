..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
IPSec Strongswan VPNaaS Driver
==========================================

https://blueprints.launchpad.net/neutron/+spec/ipsec-strongswan-driver


Problem Description
===================

Ubuntu supports strongSwan in main as of release 14.04. This driver
will provide the choice for the customers to run strongSwan on it.

Proposed Change
===============

strongSwan driver is very similar with openswan driver in addition to
quite difference of their configuration files.

So the currently implemented methods are:

* We'd have to create a strongswan_opts based off openswan_opts.

* Provide different configuration file template.

* Create a StrongSwanProcess class based off OpenSwanProcess in the
  file neutron/services/vpn/device_drivers/ipsec.py (openswan uses pluto
  and whack, while strongSwan uses 'charon' and 'stroke' respectively).

* The IPsecDriver._update_nat looks like it sets the right iptables
  ipsec needed rules for strongSwan.

Data Model Impact
-----------------

None.


REST API Impact
---------------

The latest strongSwan 5.x has different attributes than the previous
version. For example, 5.x has abandoned some configurations like
plutostart, nat_traversal, virtual_private, pfs etc, and some
configurations also have the default value like strictpolicy=no,
charonstart=yes.

OpenSwan has more similiar attributes with the previous version of
strongSwan 5.x, but not with strongSwan 5.x. Initial efforts only
support 5.x and implement an equivalent psk net-to-net vpn service
based on recommended configuration in the link [5] just as openSwan
did in the past. Future blueprints will extend other features for
strongSwan, like API, auth modes, roadwarrior-to-net etc.

So the capabilites provided by this initail implementation of the
strongSwan driver are the same with openSwan driver [6]:

* Net-to-Net Private Network connecting two private networks.

* Multiple VPN connections per tenant.

But the parmeters are somewhat different, like:

* only supporting IKEv2 policy, not support IKEv1.

* only supporting default IPSec policy and DPD now, future blueprints
  will extend for more auth modes and more encryption algorithms.

Therefore, the resources API (service, ikepolicy, ipsecpolicy,
ipsec-site-connection) will also do the corresponding code adjustment.

Security Impact
---------------

None.


Notifications Impact
--------------------

None.


Other End User Impact
---------------------

User will need to configure the INI file for the strongSwan driver.


Performance Impact
------------------

No effect to the VPNaaS performance.


IPv6 Impact
-----------

None


Other Deployer Impact
---------------------

None.


Developer Impact
----------------

None.


Community Impact
----------------

None.


Alternatives
------------

Other alternatives will be lack of community support.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Zhang Hua <joshua.zhang@canonical.com>

Work Items
----------

* StrongSwanProcess code in neutron/services/vpn/device_drivers/ipsec.py
* Work out a configuration file for best practice
* Unit tests & Advanced Service tests
* A netns wrapper to support running strongSwan in different namespace.
* Update API documentation to reflect strongSwan capabilites.
* Update user documentation to indicate how to use strongSwan option.

Dependencies
============


Testing
=======

* Unit tests
* Advanced Service tests
* Functional tests

Tempest Tests
-------------

Not applicable. use advanced service tests to cover.


Functional Tests
----------------

New neutron functional tests will be added to cover below scenario.

* new a functional test named test_vpnagent_create_process
* overide the configuration item vpn_device_driver=
  neutron.services.vpn.device_drivers.ipsec.StrongSwanDriver
* invoke create_process method then to check if ipsec process has been
  started and strongSwan configuration file has been created correctly.


API Tests
---------

Not applicable.


Documentation Impact
====================

User Documentation
------------------

The default vpn_device_driver is still openSwan, so need to update
vpn_device_driver to use strongSwan in the file /etc/neutron/vpn_agent.ini
in addition to installing strongSwan package.
vpn_device_driver=neutron.services.vpn.device_drivers.ipsec.StrongSwanDriver

API document mentioned above should also be updated, as part of this effort.

Developer Documentation
-----------------------

None.


References
==========

* [1] IPSec strongswan driver code: https://review.openstack.org/#/c/100791/

* [2] IPSec openswan driver bluprint:
  https://blueprints.launchpad.net/neutron/+spec/ipsec-vpn-reference

* [3] IPSec openswan driver code: https://review.openstack.org/#/c/33148/

* [4] IPSec openswan driver spec:
  https://docs.google.com/presentation/d/1uoYMl2fAEHTpogAe27xtGpPcbhm7Y3tlHIw_G1Dy5aQ/edit

* [5] http://www.strongswan.org/uml/testresults/ikev2/net2net-psk/

* [6] http://docs.openstack.org/api/openstack-network/2.0/content/vpnaas_ext.html

