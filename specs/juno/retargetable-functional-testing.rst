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


Problem description
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


Proposed change
===============

The goal is to introduce the concept of a 'retargetable' functional
api test.  Such a test will target an abstract client class, and by
varying the implementation of the client, the test can target multiple
backends.  One backend will be the programmatic plugin API, and
another will be the Tempest REST client for the Neutron API, so that
tests could be written and run directly against the plugin API (taking
less time than the previous REST-targeting tests) , and then be
configured to run against a live deployment with minimal effort.


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

Developers and reviewers will have to be educated as to this new
strategy for implementing API tests.  It will no longer be acceptable
to accept API changes without corresponding test additions, and there
will need to be a procedure to copy API tests into Tempest once a
given API has stabilized or tests have been modified in a
backwards-compatible way.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  maru


Work Items
----------

 1. Implement a framework for retargetable testing that proves the
    concept of writing tests to an abstraction that can support
    targeting both the plugin api and the tempest rest client.

 2. Define a procedure for transfering responsibility for stablized
    API tests to Tempest.


Dependencies
============

The testscenarios library, already listed in the
openstack/requirements repo, will become a test dependency of Neutron
as a result of this spec being implemented:

https://pypi.python.org/pypi/testscenarios/


Testing
=======

None



Documentation Impact
====================

None


References
==========

Etherpad for summit session arguing for project-maintained API tests:
https://etherpad.openstack.org/p/juno-qa-functional-api
