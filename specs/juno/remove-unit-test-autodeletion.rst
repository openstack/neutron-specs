..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Remove the use of resource autodeletion in unit tests
=====================================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/remove-unit-test-autodeletion

This blueprint describes the rational for removing resource
autodeletion in the unit tests.


Problem description
===================

The Neutron unit tests currently decorate resource creation methods
with contextlib.contextmanager to allow resources to be automatically
deleted when they are no longer needed.  This strategy is a legacy of
a time when this explicit cleanup was required to ensure isolation
between tests, but is now unnecessary since db state is automatically
thrown away after each test.  The strategy has the following negative
impacts on much of Neutron's unit test suite:

 - an execution penalty, since deletion of every created resource is
   always done and is performed through the wsgi layer
 - a debugging penalty, since contextmanagers are generators with the
   consequent loss of stack context
 - a readability penalty, since the test code is riddled with
   unnecessary 'with' blocks around resource creation
 - a coverage penalty, since implicit deletion testing is likely to
   hide gaps in testing of resource deletion


Proposed change
===============

Resource creation methods should be modified to not perform
autodeletion, as follows:

- remove the contextlib.contextmanager decorator
- change 'yield' statements to 'return'
- remove the do_delete or no_delete parameter
- remove the conditional block that deletes the resource(s)

In addition, all clients of the resource creation methods should be
updated to no longer pass a do_delete/no_delete parameter and perform
a regular call (e.g. create_resource()) rather than creating a with
block (e.g. with create_resource():).  It also may make sense to
update the names of resource creation methods, since they are often of
the form 'resourcename' rather than 'create_resourcename'.

Care will have to be taken to ensure that explicit tests for deletion
are added if they do not already exist for a given resource type.
Autodeletion ensured that any resource whose creation was tested also
had its deletion validated to some extent, but this validation will
have to be explicit from now on.  This is arguably desirable, since
relying on the 'magic' of autodeletion to provide test coverage can
potentially lull developers into a false sense of security.

There may also be tests that create multiple resources in a way that
depended on autodeltion to prevent uniqueness constraint violation.
It will be necessary to perform explicit deletion in those cases where
removal of autodeletion provokes test failures.


Alternatives
------------

The alternative to not removing autodeletion is continuing to live
with the penalties it imposes.


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

Removal of autodeletion should have a positive impact on the execution
time of the unit test suite.


Other deployer impact
---------------------

None


Developer impact
----------------

Developers and reviewers familiar with the current
contextmanager-based model of resource creation in unit tests will
have to be educated as to the move to simpler creation methods and the
need for explicit testing of deletion.

Once concern voiced at the removal of the contextmanager decorator was
that it would disallow the convenience of managing the lifecycle of
multiple resources via contextlib.nested.  If necessary, this
capability can be replicated by creating a helper method that accepts
a list of creation functions and ensures that the resources created
are cleaned up.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kevinbenton

Other contributors:
  marun

Work Items
----------

 1. Add explicit tests for deletion for resource types that have
    previously relied on autodeletion to perform implicit testing of
    deletion.

 2. Disable autodeletion in each resource creation method
    (e.g. NeutronDbPluginV2TestCase.network) in the unit test suite
    and fix any test failures that result.

 3. Remove support for autodeletion (parameters, deletion calls) from
    the resource creation methods and update their callers.

 4. Remove the contextmanager decorator from resource creation
    methods, rename the creation methods (e.g. network ->
    create_network), and update their clients.


Dependencies
============

None


Testing
=======

None


Documentation Impact
====================

None


References
==========

None
