..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Add support for retargetable functional testing
===============================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/retargetable-functional-testing

This blueprint describes the rational for adding support for
retargetable functional testing to the Neutron test suite.


Problem Description
===================

The current Neutron unit test suite contains a large number of tests
that are actually functional.  Such tests exercise almost the full
application stack through the Neutron REST API regardless of whether
that is the appropriate level of abstraction to be writing to.
Writing to the incorrect level of abstraction in this manner can
result in excessive test implementation, maintenance, and execution
costs.

There is also considerable duplication between the API-targeting tests
in the Neutron tree and the Neutron API tests managed by the Tempest
project.  This results in a rough doubling of the cost of testing a
given API with no increase in effectiveness.

The fact that formal API testing is left to Tempest has the added cost
of delaying some test additions until after a feature has been merged.


Proposed Change
===============

The goal is to introduce the concept of a 'retargetable' functional
api test.  Such a test will target an abstract client class, and by
varying the implementation of the client, the test can target multiple
backends.  One backend will be the programmatic plugin API, and
another will be the Tempest REST client for the Neutron API, so that
tests could be written and run directly against the plugin API (taking
less time than the previous REST-targeting tests) , and then be
configured to run against a live deployment with minimal effort.

Ultimately, this change enables the effort of moving functional tests
out of Tempest and into Neutron.

Alternatives
------------

None


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

Developers and reviewers will have to be educated as to this new
strategy for implementing API tests.  It will no longer be acceptable
to allow API changes without corresponding test additions.

Community Impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Maru Newby


Work Items
----------

 1. Implement a framework for retargetable testing that proves the
    concept of writing tests to an abstraction that can support
    targeting both the plugin api and the tempest rest client.

 2. Define a procedure for ensuring that tests for stable APIs are not
    changed in the same changeset as the APIs themselves are changed to
    minimize the potential for backwards-incompatible API changes.

Dependencies
============

None.

Testing
=======

Tempest Tests
-------------

Tests that were usually targeting Tempest can now be directly contributed
to Neutron. However there is a distinction between those that required
a fully fledged environment (like the scenario tests that spin up VM's and
test connectivity), and the ones that are purely targeting the Neutron API's.
The scenario tests will continue to stay in Tempest.

API Tests
---------

API tests will be contributed directly to Neutron, so that they can be
validated by the Neutron API job (https://review.openstack.org/82226).

Functional Tests
----------------

Functional tests will be contributed directly to Neutron, so that they can
be validated by the Neutron functional job (https://review.openstack.org/66967).


Documentation Impact
====================

User Documentation
------------------

The end user is not impacted.

Developer Documentation
-----------------------

The codebase should suffice in demonstrating what the approach to be
following is like, however, in-tree developer documentation on how to
maintain the functional api tests should also be contributed.

References
==========

* https://etherpad.openstack.org/p/kilo-crossproject-move-func-tests-to-projects
* https://etherpad.openstack.org/p/juno-qa-functional-api
