..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
ofagent: port monitoring w/o ovsdb accesses
===========================================

https://blueprints.launchpad.net/neutron/+spec/ofagent-port-monitor

Problem description
===================

ofagent currently scans ovsdb (via ovs-vsctl command) to get a list of ports.
ovsdb is not likely available for other openflow switch implementations.

Proposed change
===============

* implement the functionality using OFPMP_PORT_DESC instead of ovsdb.

* (optional) use OFPT_PORT_STATUS asynchronous messages to
  avoid periodic polling.

* as there is no pure openflow equivalent for port external-ids,
  we plan to use port name to identify devices. (as linuxbridge does)

Alternatives
------------

* implement switch-specific methods for every switch implementations

* use of-config
  at a glance of the spec, it doesn't seem usable at this point, though.

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

in case asyncronous messages are used, performance might be improved.

Other deployer impact
---------------------

none

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

none

but some of bug fixes need to be merged to make this useful.
for example, https://review.openstack.org/#/c/88224/

Testing
=======

ryu/ofagent third party testing would find regressions.

Documentation Impact
====================

none

References
==========

* https://www.opennetworking.org/sdn-resources/onf-specifications/openflow
* OpenFlow 1.3.3 7.3.5.7 Port Description
* OpenFlow 1.3.3 7.4.3 Port Status Message

