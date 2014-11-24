..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Cisco VPNaaS with in-band Cisco CSR router
==========================================

https://blueprints.launchpad.net/neutron/+spec/cisco-vpnaas-and-router-integration

Enhance the Cisco IPSec site-to-site VPNaaS solution, by integrating it with
a Cisco Cloud Services Router (CSR) running as a Neutron router. This allows
easy configuration of a site-to-site connection using a dynamically created
router making it practical for production use.


Problem Description
===================

In the current Proof of Concept Cisco VPNaaS, a Cisco CSR VM runs
out-of-band from OpenStack, and in parallel with a reference Neutron router.
The Cisco CSR is started manually, and independently of OpenStack. The router
is statically provisioned and information on the Cisco CSR is stored in an
.ini file for use by the Cisco VPNaaS driver.

When a VPN IPSec site-to-site connection is established, the VPNaaS drivers
use the .ini information [1] to communicate with the Cisco CSR and configure
the VPN IPSec site-to-site connection. A packet redirect is configured on the
Neutron router, to send all packets for the remote end, to the Cisco CSR.

The issues with this are:
* Cisco CSR is manually started and provisioned for use.
* The .INI file must be manually updated (error prone).
* We are effectively using two routers to provide VPNaaS capability.


Proposed Change
===============

With a previous blueprint [1], the Cisco VPNaaS driver was modified to
obtain router information from an .INI file at the time of **use**, rather
than at startup, allowing a manual way to dynamically configure VPNaaS.

This blueprint is a follow up refactoring of the driver, eliminating the need
for the .INI file, and fully automating the dynamic creation of site-to-site
connections. This makes the solution practical for operators to use in
production.

To do this, it makes use of the newly added L3 router plugin, which handles
the creation and initial provisioning of the Cisco CSR router, and contains
all the needed router information.

The VPN service driver directly calls the L3 router plugin to obtain the
management IP, username, password, inner and outer interface names, and VRF
for the router.

This has the following advantages:
* CSR is automatically created by the L3 router plugin (vs manual startup)[2].
* No longer need two routers for IPSec connection.
* No longer need .INI for router information (obtain from router plugin).
* Can dynamically create IPSec site-to-site connections.

The user would simply create a CSR router, and then select that as the router
for VPNaaS configurations.


Data Model Impact
-----------------

None.


REST API Impact
---------------

None.


Security Impact
---------------

None.


Notifications Impact
--------------------

None.


Other End User Impact
---------------------

Eliminates the need for operator to manually start and provision the Cisco CSR
and create the .ini file.


Performance Impact
------------------

Wall clock time to create a VPN connection improves, as the Neutron commands
will take all needed actions (no manual INI file changes needed).


IPv6 Impact
-----------

This is expected to work in an IPv6 environment.


Other Deployer Impact
---------------------

No longer need to manually create a Cisco CSR out-of-band for use with VPNaaS.


Developer Impact
----------------

None.


Community Impact
----------------

This completes incorporation of a Cisco based VPNaaS solution for Neutron
that is in line with the reference implementation, instead of a bolt-on
solution used by the current proof-of-concept implementation.


Alternatives
------------

There is no alternative that will give an automated, dynamic, and scalable
solution. The current mechanism, provides a proof of concept solution, but
fails to meet the needs of this spec, due to the manual interaction required.

A follow on blueprint will work towards integrating the VPN drivers with the
L3 Config Agent to reduce resource requirements. This blueprint is a step
towards that "evolution".


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
* Create methods in L3 router plugin to provide the router info needed.
* Modification of the device driver to use the passed information, instead of
  .ini file info.
* Update unit tests to reflect changes made.


Dependencies
============

None. All required components are already up-streamed.


Testing
=======

Unit tests will be updated accordingly.

Tempest Tests
-------------

No changes needed as is refactoring of existing implementation.

Functional Tests
----------------

No changes needed as is refactoring of existing implementation.


API Tests
---------

Not applicable.


Documentation Impact
====================

There are no changes to the Openstack documentation for this blueprint.
The vendor deployment/install documentation will be updated (mostly to
remove many steps).


User Documentation
------------------

None.


Developer Documentation
-----------------------

Not applicable.


References
==========
* [1] https://blueprints.launchpad.net/neutron/+spec/cisco-vpnaas-with-cisco-csr-router
* [2] https://blueprints.launchpad.net/neutron/+spec/cisco-routing-service-vm
* [3] Out-of-band VPN setup: http://docwiki.cisco.com/wiki/Install_and_Setup_of_Cisco_Cloud_Services_Router_(CSR)_for_OpenStack_VPN
