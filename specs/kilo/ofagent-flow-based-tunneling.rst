..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
OFAgent: Flow-based tunneling
=============================

https://blueprints.launchpad.net/neutron/+spec/ofagent-flow-based-tunneling

make ofagent use flow-based tunneling rather than
the current port-based tunneling.

Problem Description
===================

ofagent creates tunnel ports for each peer nodes.
It's unscalable and complex.

Proposed Change
===============

use flow-based tunneling, using tun_ipv4_src/tun_ipv4_dst NXMs.
(in addition to tun_id OXM, which is currently used by ofagent)

Note: while the use of NXMs contradicts to the one of goals of
ofagent, i.e. being portable to other switch implementations,
it isn't a problem right now because:

- the tunneling support is OVS-dependent anyway

- i've heard that the future versions of OpenFlow aims to the same direction.
  there seems to be no publically available reference for this, though.

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

none

Developer Impact
----------------

code would get simpler and easier to maintain.

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

* tweak L2populationRpcCallBackTunnelMixin api so that it can handle
  the "a single tunnel port for many networks" situation more naturally

* tweak how ofagent sets up tunnel ports

* tweak ofagent flows accordingly

Dependencies
============

none

Testing
=======

Tempest Tests
-------------

ideally multi-node testing is necessary.
however i don't plan to cover it by this blueprint.

Functional Tests
----------------

none

API Tests
---------

no api change thus no additional tests.

Documentation Impact
====================

User Documentation
------------------

should document new requirement (ryu>=3.15) for relevant NXMs support

Developer Documentation
-----------------------

none

References
==========

the current implementation of this blueprint:

- https://review.openstack.org/#/c/130676/
- https://review.openstack.org/#/c/130677/

as far as i know, NXMs are only documentated in Open vSwitch
source code:

- https://github.com/openvswitch/ovs/blob/e9bbe84b6b51eb9671451504b79c7b79b7250c3b/lib/meta-flow.h#L335
- https://github.com/openvswitch/ovs/blob/e9bbe84b6b51eb9671451504b79c7b79b7250c3b/lib/meta-flow.h#L353
