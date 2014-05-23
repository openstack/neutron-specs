=====================================
Update MidoNet plugin in Juno release
=====================================

https://blueprints.launchpad.net/neutron/+spec/midonet-plugin-juno

Update MidoNet plugin in Juno release.

Problem description
===================

MidoNet API has changed recently to be more compatible with the Neutron API, and for Juno,
the MidoNet plugin needs to be updated to utilize the new API.
Furthermore, the previous MidoNet API will soon be deprecated, which will make the current
MidoNet plugin upstream incompatible with the API.


Proposed change
===============

We will update the plugin so it works with new MidoNet backend API.
Main goal of this spec is to achieve feature parity on the current plugin.

We will re-rewrite all the handlers of the following Neutron API, which we
support today so they conform to the new MidoNet backend API.

Supported APIs:

* Network
* Subnet
* Port
* Router
* Floating IPs
* Security Group

We will submit reviews in resource by resource basis so patchsets
will be smaller and reviewable.


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
The new backend API supports backward compatibility at least in this transition period,
so the transition from the old plugin to the new one will be seamless and should not
introduce any problems to the Neutron developers.

Implementation
==============

For each of the features listed in the Proposed Change section,
all the CRUD APIs will be implemented by calling the new MidoNet API

Assignee(s)
-----------

Primary assignee:
  tomoe

Other contributors:
  joe-6
  ryu-midokura

Work Items
----------
See Implementation section.

Dependencies
============

Testing
=======

No new feature will added, so the existing unit tests and tempest tests (API and scenario) should cover everything.
However, if we come across any missing unit tests for the features proposed, we will make sure to add them.

Documentation Impact
====================
None

References
==========
None
