..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
ofagent l2pop support
==========================================

https://blueprints.launchpad.net/neutron/+spec/ofagent-l2pop

implement l2pop support for ofagent agent.

Problem description
===================

* many packets sent to tunnels because of:
    * tenant unawareness; no need to forward a packet to nodes
      which doesn't run endpoints (typically VMs) belonging to the tenant.
    * broadcasts, namely arp requests

see `L2population_blueprint`_ for more discussions.

.. _L2population_blueprint: https://wiki.openstack.org/wiki/L2population_blueprint

Proposed change
===============

* implement l2pop-based tunnel management as it's done in ovs

* implement local arp responder by handling packet-ins

Alternatives
------------

local arp responder part can be implemented differently.
for example, using nicira extensions as described in `Ovs-flow-logic`_.
however it isn't a choice for ofagent as there's no way to implement such
a flow-based arp responder without vendor extensions.

.. _Ovs-flow-logic: https://wiki.openstack.org/wiki/Ovs-flow-logic#OVS_flows_logic_with_local_ARP_responder

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

this would naturally improve performance.

see `L2population_blueprint`_ for more discussions.

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

* implement l2pop-based tunnel management as it's done in ovs.
  the following is working implementation of this.

  https://review.openstack.org/#/c/87440/

* implement local arp respoder by handling packet-ins
  the following is working implementation of this.

  https://review.openstack.org/#/c/94183/

Dependencies
============

none

Testing
=======

ryu/ofagent third party testing would find regressions.

Documentation Impact
====================

none

References
==========

* https://blueprints.launchpad.net/neutron/+spec/l2-population
* https://wiki.openstack.org/wiki/L2population_blueprint
* https://wiki.openstack.org/wiki/Neutron/OFAgent/Todo
* https://wiki.openstack.org/wiki/Neutron/OFAgent/FlowTable
* https://github.com/yamt/ryu/blob/arp-proxy/ryu/app/arp_proxy.py
