..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================================================
OFAgent: Factor out switch implementation dependent code to sub-drivers
=======================================================================

https://blueprints.launchpad.net/neutron/+spec/ofagent-sub-driver

Problem Description
===================

while ofagent aims to support plain openflow switches,
some necessary operations like tunnel port creation is not covered
by standards (yet?) and thus will be switch implementation dependent
for the foreseeable future.

Proposed Change
===============

- factor out OVS-dependent code into a sub-driver.

- introduce an option to specify a sub-driver.  its default will be OVS.

- OVS will likely be the only driver introduced as a part of this BP.
  see "Work Items" section.

Alternatives
------------

none

Data Model Impact
-----------------

none

REST API Impact
---------------

none

Security Impact
---------------

none

Notifications Impact
--------------------

none

Other End User Impact
---------------------

none

Performance Impact
------------------

none

IPv6 Impact
-----------

none

Other Deployer Impact
---------------------

a deployer who wants to use non-OVS driver needs to specify it
via the newly introduced option.  one who wants to use OVS can
just leave it default.

Developer Impact
----------------

none

Community Impact
----------------

none

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  yamamoto

Other contributors:
  kakuma

Work Items
----------

see "Proposed Change" section.

some research about non-OVS switches is necessary:

- LINC doesn't match to some of ofagent's assumptions.
  e.g. how ports are named

- lagopus doesn't seem to support dynamic add/remove of ports.  (yet?)

- IVS might be a good candidate.  it already seems to have nova/neutron
  interface drivers.

Dependencies
============

none

Testing
=======

Tempest Tests
-------------

none

Functional Tests
----------------

none

API Tests
---------

none

Documentation Impact
====================

User Documentation
------------------

The new option to specify sub driver needs to be documented.

Developer Documentation
-----------------------

none

References
==========

none
