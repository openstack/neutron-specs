..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add REJECT action rule for fwaas
==========================================

https://blueprints.launchpad.net/neutron/+spec/fwaas-reject-rule

Add REJECT into action rule of FWaaS.
Action rule of current FWaaS contains only ALLOW/DENY. DENY simply discards
the data without a response, but REJECT returns a response.
Connection source by this response can be judged to be "connection was
refused".


Problem Description
===================

Action rule of current FWaaS contains only ALLOW/DENY.
DENY simply discards the data without a response, but REJECT returns a
response.
Without REJECT feature, end users cannot know whether their accesses are
super late or rejected. This REJECT feature will be a good option for FWaaS.


Proposed Change
===============

Add REJECT into action rule of FWaaS.
Connection source by this response can be judged to be "connection was
refused".


Data Model Impact
-----------------

The db schema will be changed as below.
* add "reject" into action column in firewall_rules table.

REST API Impact
---------------

Add REJECT into action rule of FWaaS.

+----------+-------+---------+---------+------------+--------------+
|Attribute |Type   |Access   |Default  |Validation/ |Description   |
|Name      |       |         |Value    |Conversion  |              |
+==========+=======+=========+=========+============+==============+
|action    |string |RW, all  |'deny'   |'allow',    |Action rule   |
|          |       |         |         |'deny', or  |              |
|          |       |         |         |'reject'    |              |
+----------+-------+---------+---------+------------+--------------+

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

IPv6 Impact
-----------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

Another project:
* Horizon

Community Impact
----------------

None.

Alternatives
------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Higuchi Toshiaki <higuchi@mxj.nes.nec.co.jp>


Work Items
----------

The work items include:

1. Implement neutron-fwaas changes.
2. Implement python-neutronclient changes for CLI.
3. Implement Horizon changes.

Dependencies
============

None.

Testing
=======

Tempest Tests
-------------

Testing will be added to firewall tests.

Functional Tests
----------------

Scenario tests will be added to validate REJECT action rule of firewall.

API Tests
---------

Testing will be added to firewall tests.

Documentation Impact
====================

Admin guide will be updated action rule of FWaaS.

User Documentation
------------------

User guide will be updated action rule of FWaaS.

Developer Documentation
-----------------------

None.

References
==========

None.

