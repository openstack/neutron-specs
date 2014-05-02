..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Faster API list responses
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/faster-list-responses

The Neutron API currently performs a policy check for each attribute of
each element in a list response; this is penalizing response times.

Problem description
===================

Slowness in API responses has been reported in a few bug reports; it has
also been observed recently with several plugins.
The root cause of this issue is that a policy engine check is performed for
every attribute of every resource returned in a response.
A list response can contain a lot of items; this is true in particular for
ports, especially when the API requests are executed with admin credentials
without filters.
In these cases the huge number of policy checks performed becomes a
non-negligible scalability limiting factor.
For a list operation, post-plugin operations (ie: policy checks) take
about 50% of the time spent in the plugin.

While this problem is not difficult to solve, its solution might require
a change which can result in a slightly different policy engine behaviour.
The API layer for list responses puts every item through policy checks to
check which ones should not be visible to the user at all.
However it also puts every attribute through a policy check to exclude those
which should not be visible to the user, such as provider attributes for
regular users.
Doing this for every resource might make sense only if the visibility of an
attribute depends on the data in the resource itself. For instance a policy
that shows port binding attributes could be defined for all the ports whose
name is "ernest".
This might appear as great flexibility, but however it does not make sense
at all. A list operation indeed should return the same set of attributes
for each resource in the response.


Proposed change
===============

Policy checks should be used to determine the list of attributes to show only
once for every response; this attribute list should then be used for every
resource to be included in the response for the list operation.

In order for this to work there should not be any attribute-level policy
which relies on the resource value; this limitation seems to be fair.

Alternatives
------------

Alternative #1:
Move attribute-level checks back in the plugin.
This was how the software worked before the Havana release.
This does not feel right at all as this will allow, in theory, to have
deployments whose API behaviour differs according to the plugin they're
running.
All the authZ work should be performed in the common API layer.

Alternative #2:
Iterate over items and run policy check only if the attribute's policy
is dependent on the resource data.
This will require a level of introspection in the policy engine that we
currently do not have and will probably never have.
Also, it does not feel right that different items in the same list
response might end up having different attributes. A list response is
like a table, with a fixed number of columns.

Data model impact
-----------------

No Data model change.


REST API impact
---------------

No API change.

Security impact
---------------

This blueprint will not change the logic of authZ processing.
However, deployments that have defined custom policies for attributes
whose evaluation result depends on the value of the resource being
analyses might be affected.

This should be documented as it might result in attributes being returned
in responses whereas the deployer's wish was to hide them.

Notifications impact
--------------------

No impact.

Other end user impact
---------------------

No further interaction.

Performance Impact
------------------

An estimated 33% improvement on list responses.
Prototype code for this blueprint has been tested with list responses
to 4,000 items.

Other deployer impact
---------------------

Deployers should be aware that attribute-level policies dependent on resurce
values (eg.: field checks) should not be performed anymore.

Developer impact
----------------

No developer impact. Limited chance of merge conflicts.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  salvatore-orlando

Work Items
----------

This will be implemented with a single, relatively small, patch.

Dependencies
============

No dependency.

Testing
=======

The code being altered is already covered by existing unit and integration
tests.
New unit tests might be defined as needed.
New integration tests should not be needed.

Documentation Impact
====================

Guidelines for managing Neutron authZ policies should be updated.

References
==========

None.
