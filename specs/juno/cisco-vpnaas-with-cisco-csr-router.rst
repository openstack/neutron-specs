..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Cisco VPNaaS with in-band Cisco CSR router
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/cisco-vpnaas-with-cisco-csr-router

Enhance the Cisco IPSec site-to-site VPNaaS solution, by integrating it with
a Cisco Cloud Services Router (CSR) running as a Neutron router.


Problem description
===================

In the current Proof of Concept Cisco VPNaaS, a Cisco CSR VM runs
out-of-band from OpenStack, and parallel to a reference Neutron router.
The Cisco CSR is started manually, and independently of OpenStack. the router
is statically provisioned and information on the Cisco CSR is stored in an
.ini file for use by the Cisco VPNaaS driver.

When a VPN IPSec site-to-site connection is established, the VPNaaS drivers
use the .ini information to communicate with the Cisco CSR and configure
the VPN IPSec site-to-site connection. A packet redirect is configured on the
Neutron router, to send all packets for the remote end, to the Cisco CSR.

The issues with this are:
* Cisco CSR is manually started and provisioned for use.
* Static configuration of all Cisco CSRs is established before Neutron startup.
* We are effectively using two routers to provide VPNaaS capability.


Proposed change
===============

A separate blueprint, cisco-routing-service-vm [2], will be providing a Cisco CSR
VM as a Neutron router, dynamically creating and provisioning the Cisco CSR
when a router specifying this type is created.

This blueprint proposes to update the Cisco VPNaaS driver to work with this
"in-band" Cisco CSR. The VPNaaS driver will obtain information (user, password,
mgmt IP, etc.) on the Cisco CSR dynamically, instead of statically from a config
file, as done currently, so that VPN IPSec connections can then be provisioned.

Combined, these two blueprints will allow automatic creation and provisioning
of Cisco CSRs, dynamic provisioning of VPNaaS connections, and eliminate the
need for a second router and packet redirection.

Specifically, in the context of VPNaaS, the user can create a CSR1kV VM based
Neutron router, and then create a VPN service with IPsec site-to-site connections,
which will use that router.

To mitigate the risk of the dependency on the Cisco Routing Service VM
blueprint [2], the VPN implementation can be phased. In the first phase, the
code would attempt to obtain information on the CSR from the L3 plugin, but
if not available, could read the config from an INI file (as done currently
in the device driver).

The user could setup a CSR out-of-band manually (as done today), udpate the
INI file, and then proceed to create the VPN service and connections.
When the cisco-routing-service-vm [2] is upstreamed, the VPN code that does
the fallback INI file reading could be removed.


Alternatives
------------

With the current out-of-band Cisco CSR, the VPNaaS driver could re-read the
.ini file whenever it changes to obtain updated router information. That
allows dynamically creating VPNaaS connections, but still requires manual
start-up and provisioning of the CSR (and use of dual routers).


Data model impact
-----------------

None.


REST API impact
---------------

None.


Security impact
---------------

None.


Notifications impact
--------------------

None.


Other end user impact
---------------------

Eliminates the need for operator to manually start and provision the Cisco CSR
and create the .ini file.


Performance Impact
------------------

No effect to the VPNaaS performance.


Other deployer impact
---------------------

Deployment becomes much easier.


Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------


Primary assignee:
  pmichali


Work Items
----------

* Removal of device driver code that reads the .ini file with Cisco CSR info.
* Modification of service driver to obtain Cisco CSR info and pass to device
  driver.
* Modification of the device driver to use the passed information, instead of
  .ini file info.
* Update unit tests to reflect changes made.


Dependencies
============

Requires the cisco-routing-service-vm blueprint implementation, which provides
the Cisco CSR as a Neutron router and manages the life-cycle of the router.


Testing
=======

Unit tests will be updated accordingly. The cisco-routing-service-vm BP will
have Tempest tests. Currently, there are no Tempest fucntional tests for
VPNaaS, but as they become available, third-party tests will be created for
the Cisco CSR implementation.


Documentation Impact
====================

None.


References
==========

* [1] Out-of-band VPN setup: http://docwiki.cisco.com/wiki/Install_and_Setup_of_Cisco_Cloud_Services_Router_(CSR)_for_OpenStack_VPN
* [2] https://blueprints.launchpad.net/neutron/+spec/cisco-routing-service-vm
* [3] https://blueprints.launchpad.net/neutron/+spec/ipsec-vpn-reference
