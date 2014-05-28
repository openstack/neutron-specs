..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Neutron OVS agent - Dont Fragment flag
==========================================

https://blueprints.launchpad.net/neutron/+spec/neutron-ovs-agent-df-gre-vxlan

Overlay network introduce an additionnal overhead. Depends on the underlying
protocol and physical transport the 'real' MTU may have strong constraints.
However, from an instance, or a cloud user point of view, this should not be
a problem. And the Ethernet MTU inside VM should be, when possible, at least
1500 bytes.

Thus when using overlay protocol -here GRE and VXLAN- that use IP, it is
possible under certain conditions to leverage the Dont Fragment -DF- bit. Once
that bit will be set to 0, it will allow encapsulated packet to be split by
the IP stack of the hypervisor. The main goal is to allow 1500 bytes MTU
overlayed network to cross 1500 byte MTU physical network.


Problem description
===================

The problem is to span overlayed network on physical network with a smaller (or
equal MTU). As said before, the main usecase is to carry 1500 bytes MTU virtual
network over 1500 bytes MTU physical network. It's mainly required on older
network part. Once the network will be upgraded, the MTU on physical adapter
will be raised, and no fragementation will happen anymore.


Proposed change
===============

Set the options:df_default option in OVS when creating VXLAN and GRE tunnels.

Alternatives
------------

Use iptables with -j DF --clear somewehere.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

IP fragmentation could impact the network efficiency and causes some
additional load on network nodes

Other deployer impact
---------------------

It will ease some deployments by being able to span virtual networks over
MTU-limited physical network.

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  pierre-rognant

Work Items
----------

The work concern only the OVS agent. No impact on other neutron component.

Dependencies
============

None.

Testing
=======

Ensure that the default behaviour remains unchanged.


Documentation Impact
====================

Add the new 'dont_fragment' flag in the documentation.


References
==========

OVS reference documentation:
* http://openvswitch.org/ovs-vswitchd.conf.db.5.pdf (p. 24)
