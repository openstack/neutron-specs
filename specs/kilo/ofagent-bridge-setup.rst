..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
OFAgent: Simplify bridge setup
==============================

https://blueprints.launchpad.net/neutron/+spec/ofagent-bridge-setup


Problem Description
===================

- ofagent uses bridge name (eg. "br-int") to specify the bridge to control.
  it's OVS-dependent.

- ofagent automatically set up the bridge on startup.
  (i.e. add-br, set br protocols, set-controller, etc)
  there are little point to perform it every time an agent starts up.
  one-time setup during deployment is enough.  also, it's OVS-dependent.

Proposed Change
===============

- use datapath-id instead of bridge name.
  introduce a new config option to specify the bridge using datapath-id.
  make it an alternative of "integration_bridge" option.

- remove code to set up the bridge from ofagent.
  document the instruction to set up the bridge instead.

- adapt devstack accordingly.

- retire "integration_bridge" option for ofagent later.
  (probably in "L" cycle.)

Alternatives
------------

instead of datapath-id, OFPMP_DESC might be usable to distinguish bridge.
OVS by default doesn't provide useful description though.

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

a deployer needs to update his config.

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

Dependencies
============

none

Testing
=======

update the existing tests if necessary.

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

the new option and upgrade instruction need to be documented.

Developer Documentation
-----------------------

none

References
==========

none
