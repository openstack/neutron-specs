..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
OFAgent: Merge br-int and br-tun
==========================================

https://blueprints.launchpad.net/neutron/+spec/ofagent-merge-bridges

merge br-int and br-tun and stop using OVS patch ports feature.
this involves drastic flow table changes.

Problem description
===================

ofagent aims to be portable among switch implementations.
currently it uses some of OVS specific features.
patch ports is one of them.

Proposed change
===============

merge br-int and br-tun into a single bridge.

Alternatives
------------

* give up and declare that tunnel support is only for OVS.  this is not
  what we want to do.

* use veth pair instead.  this is not ideal as it still requires
  multiple logical bridge feature.  besides that, it likely involve
  some performance loss because patch ports is better optimized than
  veth pair.

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

because OVS patch ports hardly have negative performace effects for fast path,
this change is not expected to improve performance.

Other deployer impact
---------------------

when upgrading the agent, deployer might want to remove br-tun.

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

* design flow table.  see `flow_table`_ for WIP design.

.. _flow_table: https://wiki.openstack.org/wiki/Neutron/OFAgent/FlowTable

* implement it in ofagent neutron agent

* document the upgrade procedure

Dependencies
============

strictly speaking, none.
but the following items are nice to have before this.

* `ofagent-l2pop`_ blueprint (our WIP implementation relies on this)

.. _ofagent-l2pop: https://blueprints.launchpad.net/neutron/+spec/l2-population

* matrohon's `get_device_details-enhancement`_.
  or other way to obtain device's mac_address.
  (for example, make l2pop provide device-id for entries.)
  we want to use it for node local routing of packets.

.. _get_device_details-enhancement: https://review.openstack.org/#/c/96181/

Testing
=======

* unit tests

* existing third party testing

Documentation Impact
====================

* document the upgrade procedure

References
==========

* WIP flow table for this blueprint
  https://wiki.openstack.org/wiki/Neutron/OFAgent/FlowTable

* WIP implementation of the above
  https://github.com/yamt/neutron/tree/ofagent-merge-bridges

