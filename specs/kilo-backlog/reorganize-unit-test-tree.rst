..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Example Spec - The title of your blueprint
==========================================

https://blueprints.launchpad.net/neutron/+spec/reorganize-unit-test-tree

Reorganize the structure of the unit test tree to be consistent with
the structure of the code tree.

Problem Description
===================

There is no consistent organization of unit test modules
(neutron/test/unit/\*).  There is no easy way to find the unit tests
for a given module, making it challenging to determine whether code is
well-tested.  There are also no clear guideline as to where new tests
should go, ensuring that the problem continues.

Proposed Change
===============

The module structure of the neutron/tests/unit subtree should be
changed to mirror the structure of the code tree.

 * The path structure should be the same.  This implies that modules
   under the path:

   neutron/[path]

   should have test modules in the following location

   neutron/tests/unit/[path]

   For example, test modules for 'neutron/scheduler' should have tests
   at 'neutron/tests/unit/scheduler'.

 * The name of the test modules should correspond to the name of the
   module under test, prefixed with 'test\_'.  For example, the module
   'neutron/scheduler/dhcp_agent_scheduler.py' implies the test module
   'neutron/tests/unit/scheduler/test_dhcp_agent_scheduler.py'.

This requirement should be documented such that new changes follow
this scheme.

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Patch authors and reviewers will need to ensure that new changes
maintain the consistent structuring of the unit test tree.

Community Impact
----------------

None

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  marun

Work Items
----------

 * Reorganize the test tree

Dependencies
============

None

Testing
=======

None

Tempest Tests
-------------

None

Functional Tests
----------------

None

API Tests
---------

None

Documentation Impact
====================

None

User Documentation
------------------

None

Developer Documentation
-----------------------

The required structure of the unit test tree should be documented in the in-tree developer documentation.

References
==========

[1] https://pytest.org/latest/goodpractises.html
