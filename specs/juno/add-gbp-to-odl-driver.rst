..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Add GBP support to ODL ML2 Driver
=================================

Launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/odl-gbp-ml2-driver

This blueprint proposes the support of the Group Based Policy extension
with the OpenDaylight ML2 mechanism driver.

Problem description
===================
The Group Based Policy blueprint has proposed new application centric APIs for
Neutron. Similar to the work in Neutron, there is work in the OpenDaylight
project to implement these APIs as well. With the APIs being implenented in
both Open Source projects, we want to propose a GBP driver to
proxy the new GBP APIs over to OpenDaylight.

Proposed change
===============
The proposed change will add a new GBP driver to support OpenDaylight.
It will implement the PolicyDriver interface as defined in the abstract base
class services.group_policy_driver_api.PolicyDrive, as documented in
the GBP BP.

The ODL GBP driver will work in conjunction with the existing ODL ML2 mechanism
driver, which would still need to be involved to take care of binding ports.

Alternatives
------------
There are no alternatives as this is simply adding support for the new Group
Based Policy APIs into the OpenDaylight MechanismDriver.

Data model impact
-----------------
None, as this change is simply adding support for new API extensions.

REST API impact
---------------
The ODL ML2 MechanismDriver will have support for the new GBP API extensions
added.

Security impact
---------------
None.

Notifications impact
--------------------
None.

Other end user impact
---------------------
Users deploying Juno with the Group Based Policy extensions will be able to
utilze these new APIs in conjunction with the Helium version of OpenDaylight.

Performance Impact
------------------
No change here.

Other deployer impact
---------------------
To utilize the GBP APIs with OpenDaylight, the following versions of software
are required:
* Neutron: Juno
* OpenDaylight: Helium

Developer impact
----------------
None.

Implementation
==============

Assignee(s)
-----------
Kyle Mestery (mestery)
Ryan Moats (regXboi)

Work Items
----------
* Add support for GBP APIs into ODL MechanismDriver.

Dependencies
============
This work is dependent on the GBP APIs being merged in Juno.

Testing
=======
Additional unit tests will be added. Further, new tempest tests for GBP will
be run in the OpenDaylight CI system.

Documentation Impact
====================
The documentation will be updated to show support for the GBP API extensions
in the ODL MechanismDriver.

References
==========
https://git.openstack.org/cgit/openstack/neutron-specs/tree/specs/juno/group-based-policy-abstraction.rst
https://wiki.opendaylight.org/view/Group_Policy:Main

