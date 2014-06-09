..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Migrate to oslo-messaging
==========================================


Problem description
===================
Neutron currently uses an older version of Oslo's RPC code which is being
deprecated in favor of oslo.messaging.

Proposed change
===============
Neutron will be updated to utilize oslo.messaging and the copied code from
oslo-incubator will be removed from the tree.

Alternatives
------------
None.  oslo.messaging is the only supported RPC framework from the Oslo team.


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
The content of the notifications should not change; however, configuration
options will slightly change.

Other end user impact
---------------------
The underlying library uses the compatible wire format, so users upgrading
should not be impacted.

Performance Impact
------------------
This is a code refactor for a new library api, so performance changes will be
minimal.

Other deployer impact
---------------------
Configuration options will change slightly to utilize the standard set from
oslo.messaging. The options are standard across other OpenStack projects, so
deployers will benefit from the standardization.


Developer impact
----------------
None.  Shims will be used to provide compatibility with internal code
interfaces.

Implementation
==============

Assignee(s)
-----------
ihrachys


Work Items
----------
- Introduce shims modules for existing RPC classes with Neutron
- Introduce exception compatibility layer
- Convert shims to use oslo.messaging constructs
- Future work could include removal of shims, but this can be done later.

Dependencies
============
None

Testing
=======
Exisiting Unit, Functional, and Tempest tests will ensure the refactor does
not introduce any regressions.  Grenade testing should validate that the
message payload are compatible.

Documentation Impact
====================
Messaging configuration will need to be updated to reflect the set of options
provided by oslo.messaging.

References
==========
N/A

