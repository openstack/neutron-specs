..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Provider Validation for L3 Plugins
==========================================

https://blueprints.launchpad.net/neutron/+spec/l3-svcs-vendor-validation

Currently, validation of end user selections is tightly bound to the
reference implementation. As a result, alternate providers cannot
validate end user configuration selections, until after they have
already been persisted.


Problem description
======================================

When an L3 service request is received by Neutron, it invokes the
corresponding service plugin, which *validates* and *persists* info
for the request (typically in one step). Finally, the plugin will
invoke the service driver to *apply* the changes.

It is at the *apply* stage, when the provider's service driver has
an opportunity to take action on the request. If the provider has
additional constraints, it can reject the request, but the
resource(s) will already have been persisted.

For example, when creating a VPN IPsec site connection using an IKE
policy that selected v2, the connection will pass validation, be
persisted, and the service driver called. With the Cisco CSR,
which currently does not support v2 via its REST API, the service
driver can raise an exception during the *apply*. However, the
connection will have already been created and persisted.

In addition, if a service driver has persistence to perform, it
must do that during the *apply* phase.


Proposed change
===============

This proposal is to break out the validation into a separate
function, and provide a way for provider service drivers to
extend or override the validation logic, if desired.

Research into the current validation shows that it must be
done during the persistence step, to handle the situation
where multiple clients may be making changes to the same
resources simultaneously. The transaction will enforce
consistency.

The approach planned is to create default validation logic
in the service driver abstract base class, and during
persistance, select the active service driver and call it's
validation method.

The service driver can perform validation before, after, or
instead of the default logic, or not provide a validation
method and the default validation in the ABC will be performed.

Goal is to roll out the same proposal for all L3 service plugins.

This will be a structural change, and not a behavorial change.

As part of the Juno summit there was a discussion of using
the TaskFlow package for managing the processing for validate,
persistence, and apply operations.

More research will be needed on TaskFlow, but regardless, the
validation logic should be separated from the persistence logic
and can be done as preliminary work.

Alternatives
------------

An alternative is to mark the resource status/state as *ERROR*, but
the database will still be inconsistent with the actual state of
the requested operation.


Data model impact
-----------------

No changes to the database schema or operations is expected, other
that it would now accurately reflect the actual state of the
resources.


REST API impact
---------------

No expected changes to REST APIs.


Security impact
---------------

None.


Notifications impact
--------------------

None?


Other end user impact
---------------------

None.


Performance Impact
------------------

There is no significant performance impact expected.


Other deployer impact
---------------------

None.


Developer impact
----------------

Providers will want to update their service drivers to take advantage
of the new ability to validate and reject requests, based on provider
capabilities.


Implementation
==============

Assignee(s)
-----------


Primary assignee:
  pmichali

Other contributors:
  Anyone

Given the scope and number of L3 services, additional contributers
would be greatly appreciated!


Work Items
----------

For each L3 service create and update API:

* Refactor the database functionality
  * Pull out validation logic and move to service driver ABC
  * Add hook to call validation logic now in service driver

* Add validation method to provider's service driver, if needed
* Update unit tests based on structural changes

Note: The separation of the validation logic can be done on
each L3 service independently (there are no dependencies), and
as needed (e.g. only on resources where multiple implementations
have different validation needs), if desired.


Dependencies
============

None


Testing
=======

Since this is a structural change, it is expected that no additional
tempest tests will be needed.

Unit tests will need modification, mostly due to method name changes,
but may also change due to better isolation of the logic. Tests that
were checking both validation and persistence (or were doing
persistence to check validation), can be split out into more isolated
test cases. This should reduce the amount of overlap in test cases
(possibly eliminating some tests), better isolation, and possibly
better performance (not having to do several tasks to test some
functionality).


Documentation Impact
====================

None.


References
==========

None at this time.
