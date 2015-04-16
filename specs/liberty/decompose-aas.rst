..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Decompose vendor plugins/drivers for neutron-aas
================================================

https://blueprints.launchpad.net/neutron/+spec/decompose-aas

Move vendor service drivers out-of-tree, starting in Liberty,
complete by the end of M. Includes ref implementations, if the Liberty
summit decides to remove neutron ref implementations from in-tree.

Problem Description
===================

As part of Kilo, neutron undertook a major effort to decompose vendor
code out-of-tree[1]. During discussion of this feature, it was decided to
allow service drivers to remain in-tree, as those repos have a smaller
community, but to eventually adopt the same model.

This proposal attempts to outline the timeline for decomposing service
drivers. This is not a purely Liberty focused spec, but instead attempts
to communicate timelines and gather consensus around this topic for the
next several cycles.

Proposed Change
===============

All new service drivers which are not in gerrit as of when this spec is
approved, must be out-of-tree shim style drivers, as show in the neutron
vendor decomp spec[1].

Existing drivers have until the end of the M cycle to convert themselves to
out-of-tree shims.

Third-party CI will be required/encouraged with the same rules as for
neutron plugins.

Affected drivers:

Lbaas drivers, v1:

* haproxy
* radware
* netscaler
* embrane
* a10networks
* vmware

Lbaas drivers, v2:

* radware
* haproxy
* a10networks
* brocade
* kemp

Firewall drivers, from neutron_fwaas/services/firewall/drivers:

* cisco
* freescale
* linux
* mcafee
* varmour
* vyatta

VPN

* openswan (deprecated, will not move)
* strongswan
* cisco
* vyatta


Data Model Impact
-----------------

None.

REST API Impact
---------------

None.

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

Same as neutron decomp impact.[1]

Performance Impact
------------------

None.

IPv6 Impact
-----------

None.

Other Deployer Impact
---------------------

Same as neutron decomp impact.[1]

Developer Impact
----------------

Same as neutron decomp impact.[1]

Community Impact
----------------

Same as neutron decomp impact.[1]

Alternatives
------------

Continue having core teams review vendor code.

Implementation
==============

Assignee(s)
-----------

Putting a name here as a starter.  Anyone else, feel free.

Primary assignee:
  https://launchpad.net/~dougwig

Other contributors:
  <launchpad-id or None>

Work Items
----------

- Communicate change for drivers in Liberty, deadlines for M.

Dependencies
============

None.

Testing
=======

Tempest Tests
-------------

Covered by third-party CI, as today.

Functional Tests
----------------

Covered by third-party CI, as today.

API Tests
---------

Covered by third-party CI, as today.

Documentation Impact
====================

User Documentation
------------------

None.

Developer Documentation
-----------------------

None.

References
==========

[1] https://github.com/openstack/neutron-specs/blob/master/specs/kilo/core-vendor-decomposition.rst
