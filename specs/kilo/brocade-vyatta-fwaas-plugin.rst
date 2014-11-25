..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Brocade Neutron FWaaS driver for Vyatta vRouter
===============================================

https://blueprints.launchpad.net/neutron/+spec/brocade-vyatta-fwaas-plugin

Introduce the Brocade Vyatta Firewall device driver to provide FWaaS solution
using Vyatta vRouter VM running as a Neutron router. The driver implements
‘Perimeter Firewall’ functionality to filter traffic between tenant private
networks and external networks.


Problem Description
===================

Brocade Vyatta vRouter is a multi-service product that provides various L3
and L4 services like Routing, NAT, Firewall, VPN, etc. While basic neutron
router L3 functions are available using Brocade Vyatta L3 plugin [1]
vRouter's Firewall functionality is currently not configurable through
existing Neutron FWaaS APIs.

Proposed Change
===============

This blueprint proposes a new vendor device-driver for Neutron FWaaS agent.
There is no change proposed in the FWaaS service plugin side as existing
reference FWaaS plugin is sufficient.

::

>>                    +----------------------+
>>                    |   Vyatta L3 NAT      |
>>                    |        Agent         |
>>                    |                      |
>>                    | +------------------+ |
>>                    | |    FWaaS Agent   | |
>>    RPC to FWaaS    | +------------------+ |
>>    service plugin  | |    Vyatta FWaaS  | |
>>                    | |   Device Driver  | |
>>    <---------------+ |                  | |
>>                    +-+--------+---------+-+
>>                               |
>>                               |
>>                               | REST API
>>                               |
>>                      +--------v---------+
>>                      |                  |
>>                      |                  |
>>                      | Vyatta vRouter   |
>>                      |                  |
>>                      |                  |
>>                      |                  |
>>                      |                  |
>>                      +------------------+


Vyatta L3 NAT agent uses Neutron L3 NAT agent to associate the firewall to
the router interfaces.

Vyatta FWaaS device driver will invoke the Vyatta vRouter REST APIs for the
below CRUD APIs as and when determined by the FWaaS agent

1. create_firewall
2. update_firewall
3. delete_firewall

All these functions are similar to the existing reference FWaaS device-driver
implementation.
Due to limitations of the existing neutron firewall plugin, firewall rules
will get applied to all the tenant routers. Also this effort will be aligned
with the community direction of the firewall insertion mode on a single
router spec[3].

Note, we are aware of the current L3 agent refactoring proposed for Kilo [4].
Given the device driver interface is planned to be kept as-is the changes
proposed in this blueprint will integrate with minimal impact vis-a-vis the
refactoring.

This effort is part of a wider set of blueprints to offer Neutron L3 and L4
services using Vyatta vRouter VM:

* [1] introduces neutron router functionality using Vyatta vRouter.
* [2] introduces VPN service using the Vyatta vRouter.


Data Model Impact
-----------------

None.

REST API Impact
---------------

None.

Security Impact
---------------

The device driver will use a common RESTapi client library that uses
basic-auth authentication to connect to Vyatta vRouter.


Notifications Impact
--------------------

None.


Other End User Impact
---------------------

When a tenant creates a Firewall using Neutron API it will be created on the
carrier-grade Vyatta vRouter.

Performance Impact
------------------

None.

IPv6 Impact
-----------
None.

Other Deployer Impact
---------------------

Operators should first configure Brocade Vyatta L3 plugin as described in [1].
Neutron firewall plugin, Vyatta L3 agent and the firewall driver should be
configured. Once configured, Vyatta FWaaS driver will be invoked for the
Firewall CRUD operations on the tenant Router.

Developer Impact
----------------

None.

Community Impact
----------------

Validating Neutron FWaaS APIs with multiple vendor, including this one from
Brocade, will help to move out of current experimental state for these APIs.

Alternatives
------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  vishwanathj

Other contributors:
  natarajk.

Work Items
----------

* Add new Vyatta firewall device driver.
* Add unit tests required to test the device driver.


Dependencies
============

* Brocade Vyatta L3 Plugin [1]


Testing
=======

Tempest Tests
-------------

- 3rd party testing will be provided (Brocade Vyatta CI).
- Brocade Vyatta CI will report on all changes affecting this plugin.
- Testing is done using devstack and Vyatta vRouter.

Functional Tests
----------------

Scenario tests will be added to validate the Vyatta FWaaS implementation.

API Tests
---------

No new API tests are planned as no APIs are changed as part of this blueprint.


Documentation Impact
====================

User Documentation
------------------

Brocade specific documentation will be updated on the availability of this
functionality in Neutron and the fwaas_device_driver configuration required
to enable it.

Developer Documentation
-----------------------

None.

References
==========

* [1] https://blueprints.launchpad.net/neutron/+spec/l3-plugin-brocade-vyatta-vrouter
* [2] https://docs.google.com/document/d/1PJaKvsX2MzMRlLGfR0fBkrMraHYF0flvl0sqyZ704tA/
* [3] https://review.openstack.org/#/c/138672/
* [4] https://blueprints.launchpad.net/neutron/+spec/restructure-l3-agent
