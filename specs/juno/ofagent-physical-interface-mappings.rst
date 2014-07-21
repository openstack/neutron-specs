..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
ofagent: physical interface mappings
====================================

https://blueprints.launchpad.net/neutron/+spec/ofagent-physical-interface-mappings

Problem description
===================

ofagent currently has "bridge_mappings" config option inherited
from ovs agent.  it assumes multiple bridges are available.
the assumption is ok for ovs agent but not desirable for ofagent.

Proposed change
===============

* introduce "physical_interface_mappings" config option for ofagent
  with a similar semantics to linuxbridge.

* deprecate bridge_mappings for ofagent later.

* document the upgrade procedure for deployers.

Alternatives
------------

no alternative i can think of right now.

Data model impact
-----------------

none

REST API impact
---------------

none

Security impact
---------------

none

Notifications impact
--------------------

none

Other end user impact
---------------------

none

Performance Impact
------------------

none

Other deployer impact
---------------------

a deployer who's using ofagent might need to tweak his configuration.

* create veth pairs for each bridges in the existing bridge_mappings

* use physical_interface_mappings instead of bridge_mappings

Developer impact
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

see "Proposed change" section.

Dependencies
============

we need to stop installing flows in physical bridges.
we plan to do that in the following.

* https://review.openstack.org/#/c/98702/

Testing
=======

ryu/ofagent third party testing would find regressions.

Documentation Impact
====================

* document the upgrade procedure for deployers.

References
==========

none
