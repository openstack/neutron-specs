..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Security group extension support in PLUMgrid plugin
====================================================

Launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/plumgrid-plugin-security-groups

Proposal to add security group extension in PLUMgrid plugin.

Problem description
===================

Currently, PLUMgrid plugin does not support security group extension. This
feature is required to support PLUMgrid security policy framework and provide
all the functionality exposed in security group API extensions today.

Proposed change
===============

Introduce Neutron security group extension in PLUMgrid plugin. This will
include support for all existing security group and rules APIs:
1) CRUD operations for security groups
2) CRUD operations for security group rules

Alternatives
------------
None

Data model impact
-----------------
None - Existing data model will suffice here.

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
PLUMgrid provides a security policy framework which offers all the
functionality exposed in security group extension APIs. Mapping of Neutron
security groups fits well in the subset of cababilities provided by PLUMgrid.

All the existing worklows supported by Neutron API[1] or python-neutron
client[2] will be suppoted.


Assignee(s)
-----------

Primary assignee:
  Fawad Khaliq <fawadkhaliq>

Other contributors:
  None

Work Items
----------

Extension code in PLUMgrid plugin
Unit test cases for PLUMgrid plugin
PLUMgrid CI update to enable security group testing

Dependencies
============
None

Testing
=======

Covered in PLUMgrid plugin unit testing
Leverage existing Tempest infra and run as part of PLUMgrid CI

Documentation Impact
====================
None

References
==========
[1] Neutron Security Groups Extension API Reference
http://docs.openstack.org/api/openstack-network/2.0/content/security_groups.html

[2] Security Groups Operations Admin Guide
http://docs.openstack.org/admin-guide-cloud/networking_adv-features.html#security-groups
