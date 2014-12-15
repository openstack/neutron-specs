..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
Brocade Vyatta VPN service and device driver for Neutron
========================================================

https://blueprints.launchpad.net/neutron/+spec/brocade-vyatta-vpnaas-plugin

Introduce the Brocade Vyatta VPN service and device driver to provide VPNaaS
solution using Vyatta vRouter VM running as a Neutron router.


Problem Description
===================

Brocade Vyatta vRouter is a multi-service product that provides various L3 and
L4 services like Routing, NAT, Firewall, VPN, etc. While basic neutron router L3
functions are available using the Brocade Vyatta L3 plugin [1] vRouter's IPSec
site-to-site VPN functionality is currently not configurable through existing
Neutron VPN APIs.

When available Cloud Service providers would be able to create site-to-site
IPSec VPN to connect tenant networks to remote DC networks using Vyatta vRouter.


Proposed Change
===============

This blueprint proposes a new vendor service and device drivers for the
Neutron VPN plugin and agent.

::


    +----------------------+                  +----------------------+
    |                      |                  |   Neutron L3 Agent   |
    |                      |                  |                      |
    |                      |                  |                      |
    | +------------------+ |                  | +------------------+ |
    | |       VPN        | |                  | |    VPN Agent     | |
    | |  Service Plugin  | |                  | +------------------+ |
    | +------------------+ |                  | |   Vyatta VPN     | |
    | |   Vyatta VPN     | |        RPC       | |  Device Driver   | |
    | | Service Driver   | + <--------------> | |                  | |
    +-+------------------+-+                  +-+--------+---------+-+
                                                         |
                                                         |
                                                         | REST API
                                                         |
                                                +--------v---------+
                                                |                  |
                                                |                  |
                                                |  Vyatta vRouter  |
                                                |                  |
                                                |                  |
                                                |                  |
                                                |                  |
                                                +------------------+




Vyatta VPN service driver will inherit from the reference ipsec service driver
except it will use a unique topic for RPCs to and from the Vyatta VPN device
driver. This is done to be inline with existing service-type framework already
partially in place and the expectation that if neutron flavor framework [4]
materializes the functionality proposed in this BP will work as-is.

Vyatta VPN device driver will perform the following functions:

1. Handles the RPC message from vpn service-plugin that indicates a CRUD
   operation for site-to-site vpn connection
2. Gets the list of VPN services from the service-plugin using a RPC call
3. Prepares the list of new, deleted and updated vpn connection based on the
   local service-cache entries
4. Processes the above lists into effect using vRouter's REST API interface
5. Updates the local service-cache to reflect the new changes
6. Reports the status of the vpn connections back to the vpn service-plugin

All these functions are similar to the existing reference vpn device driver
implementation.

Additionally during L3 Agent startup the device driver will read vRouter VPN
configuration using its REST API to rebuild the local service-cache. Once
rebuilt the steps 2 through 6 are repeated. This helps to bring the vRouter
VPN configuration to be in sync with the changes (if any) in the plugin DB
while the L3 agent was down.

Note, we are aware of the current L3 agent refactoring proposed for Kilo [3].
Given the device driver interface is planned to be kept as-is the changes
proposed in this blueprint will integrate with minimal impact vis-a-vis the
refactoring.

This effort is part of a wider set of blueprints to offer Neutron L3 and L4
services using the Vyatta vRouter VM:

* [1] introduces neutron router functionality using the Vyatta vRouter
* [2] introduces firewall service using the Vyatta vRouter.


Data Model Impact
-----------------

None.

REST API Impact
---------------

None.

Security Impact
---------------

The device driver will use a common RESTapi client library that uses basic-auth
authentication to connect to Vyatta vRouter.


Notifications Impact
--------------------

None.


Other End User Impact
---------------------

When tenants creates VPN using the Neutron API it will be created on the
carrier-grade Vyatta vRouter.

Performance Impact
------------------

None.

IPv6 Impact
-----------

Expected to work with IPv6


Other Deployer Impact
---------------------

Operators should first configure the Brocade Vyatta L3 plugin as described in
[1]. Then they can configure the new vpn service and device drivers to offer
Vyatta VPN using Neutron APIs as follows:

* Edit /etc/neutron/neutron.conf and specify Vyatta VPN service driver as the default service provider for VPN.

::

>> [service_providers]
>> service_provider=VPN:brocade:neutron.services.vpn.service_drivers.vyatta_ipsec.BrocadeVyattaIPsecVPNDriver:default


* Edit /etc/neutron/vpn_agent.ini and specify Vyatta VPN device driver.

::

>> [vpnagent]
>> vpn_device_driver=neutron.services.vpn.device_drivers.vyatta_ipsec.VyattaIPSecDriver


Developer Impact
----------------

None.

Community Impact
----------------

Validating Neutron VPN APIs with multiple vendor, including this one from
Brocade, will help to move out of current experimental state for these APIs.

Alternatives
------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  srics-r

Other contributors:
  None

Work Items
----------

* Add new vyatta service driver for VPN service plugin
  (currently planned for neutron/services/vpn/service_drivers/vyatta_ipsec.py)
* Add new vyatta device driver for VPN agent
  (currently planned for neutron/services/vpn/device_drivers/vyatta_ipsec.py)
* Add unit tests required to test the new code
* Add tempest tests for new scenarios


Dependencies
============

* Brocade Vyatta L3 Plugin [1]


Testing
=======

Tempest Tests
-------------

- 3rd party testing will be provided (Brocade Vyatta CI)
- Brocade Vyatta CI will report on all changes affecting this plugin
- Testing is done using devstack and Vyatta vRouter

Functional Tests
----------------

None

API Tests
---------

No new API tests are planned as no APIs are changed as part of this blueprint.


Documentation Impact
====================

None.

User Documentation
------------------

Brocade specific documentation will be updated on the availability of this
functionality in Neutron and the vpn_device_driver configuration required to
enable it.

Developer Documentation
-----------------------

None.

References
==========

* [1] https://blueprints.launchpad.net/neutron/+spec/l3-plugin-brocade-vyatta-vrouter
* [2] https://blueprints.launchpad.net/neutron/+spec/firewall-plugin-for-brocade-vyatta-vrouter
* [3] https://blueprints.launchpad.net/neutron/+spec/restructure-l3-agent
* [4] https://review.openstack.org/#/c/102723
