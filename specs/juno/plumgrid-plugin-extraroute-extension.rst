..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Extra route extension support for PLUMgrid Plugin
=================================================

Launchpad Blueprint:
https://blueprints.launchpad.net/neutron/+spec/plumgrid-plugin-extraroute-extension

Proposal to add support for extra route extension in PLUMgrid plugin

Problem description
===================
Currently, PLUMgrid plugin does not support Neutron extra-route extension.
This feature is required to support static routes in PLUMgrid routers.

Proposed change
===============
Adding extension support code in PLUMgrid plugin

Alternatives
------------
None

Data model impact
-----------------
None

REST API impact
---------------
None

Security impact
---------------
None

Notifications impact
--------------------
None

Other end user impact
---------------------
None

Performance Impact
------------------
None

Other deployer impact
---------------------
None

Developer impact
----------------
None

Implementation
==============
PLUMgrid supports adding static routes to PLUMgrid routers which fits nicely
with extra route extension supported by Neutron.

Assignee(s)
-----------

Primary assignee:
  Fawad Khaliq <fawadkhaliq>

Other contributors:
None

Work Items
----------
Extension code in PLUMgrid plugin
Unit test coverage in PLUMgrid plugin


Dependencies
============
None

Testing
=======

Unit Test coverage for extra-route extension within PLUMgrid unit tests

Documentation Impact
====================
None

References
==========
N/A
