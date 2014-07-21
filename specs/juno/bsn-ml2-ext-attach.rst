..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================
Big Switch - ML2 external attachment for vlan_switch type
=========================================================

https://blueprints.launchpad.net/neutron/+spec/bsn-ml2-ext-attach

This spec is to add the ability for the Big Switch ML2 driver to
configure external attachment points for the vlan_switch type.


Problem description
===================

Once the external attachment point extension[1] is implemented, ML2
drivers will have the ability to bind external attachment points to
neutron networks. Since the Big Switch backend has full control over
the fabric, it can bind physical ports to neutron networks. However,
the code needs to be added in the ML2 driver to take these calls and
pass them to the backend controller.



Proposed change
===============

Add the appropriate external attachment methods to the Big Switch ML2
driver to relay the information to the backend controller.


Alternatives
------------
N/A


Data model impact
-----------------
N/A


REST API impact
---------------
N/A


Security impact
---------------
N/A


Notifications impact
--------------------
N/A

Other end user impact
---------------------

N/A

Performance Impact
------------------

N/A

Other deployer impact
---------------------

N/A

Developer impact
----------------

N/A


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kevinbenton

Work Items
----------

* Add the relevant methods to the Big Switch ML2 driver to receive the external
  attachment calls and relay them to the backend controller.
* Add unit tests

Dependencies
============

Implementation of external attachment point in the ML2 plugin.[1]

Testing
=======

Unit tests until a physical testbed can be setup in the 3rd party CI.


Documentation Impact
====================

N/A

References
==========

1. https://github.com/openstack/neutron-specs/blob/master/specs/juno/neutron-externa    l-attachment-points.rst
