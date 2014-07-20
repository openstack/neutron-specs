..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
ML2 Mechanism Driver for Snabb NFV
==================================

https://blueprints.launchpad.net/neutron/+spec/snabb-nfv-mech-driver

We propose to add Neutron support for Snabb NFV, an open source
Network Functions Virtualization system.


Problem description
===================

Snabb NFV supports Internet Service Providers who:

1. Want to deploy core network equipment with virtual machines.
2. Require Neutron and Virtio-net software abstractions.
3. Demand N x 10G performance with diverse traffic workloads.
4. Want to keep their OpenStack deployment fully open source.

Snabb NFV is based on a user-space dataplane in the style of Intel
DPDK (Snabb Switch).


Proposed change
===============

To add a "Snabb NFV" mechanism driver that performs port binding for
Neutron ports that are to be implemented using the Snabb NFV
networking stack on compute hosts.

The mechanism driver will only implement port binding callbacks. The
port binding logic will assign selected ports to VIF_VHOSTUSER and
assist with scheduling them onto suitable hosts and ports.

This mechanism driver is not responsible for synchronizing the Neutron
configuration with the agent on the compute host. That task is
delegated to a separate (out-of-tree) daemon that runs on the database
node and distributes snapshots of the Neutron configuration to the
Snabb NFV agent on each compute node. Thus the mechanism driver is
able to assume that the contents of the Neutorn database is available
on all nodes.

(See reference below for links to the out-of-tree mechanism for
snapshotting and distributing Neutron the database contents.)

Alternatives
------------

This blueprint is designed to minimize the amount of new code that has
to be introduced into the Neutron tree.

One alternative would be to bring all or part of the Snabb NFV
software directly into the Neutron tree. However, OpenStack and Snabb
NFV are likely to benefit from being developed separately, as with
other projects including OpenDaylight, OVS, and OVS-DB.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Snabb NFV will provide higher packet rates than standard kernel
networking.


Other deployer impact
---------------------

Snabb NFV requires compatible versions of Libvirt, QEMU, and Snabb
Switch to be installed on the compute hosts. Compute hosts must also
have network hardware that is supported by Snabb Switch.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
Luke Gorrie <lukego>

Other contributors:
Nikolay Nikolaev <n-nikolaev>

Work Items
----------

1. ML2 mech driver with portbinding for creating VIF_VHOSTUSER ports.
2. Configuration support to select which ports to bind to this driver.

Dependencies
============

1. Nova VIF_VHOSTUSER driver support: https://blueprints.launchpad.net/nova/+spec/vif-vhostuser

Testing
=======

The code will be covered by unit tests.
Third party tempest tests will be provided for CI.

Documentation Impact
====================

Configuration Reference guide will be updated from the code.

References
==========

* Snabb NFV: http://snabb.co/nfv.html

* vhost-user: http://www.virtualopensystems.com/en/solutions/guides/snabbswitch-qemu/

* Deutsche Telekom TeraStream project (first Snabb NFV user): http://blog.ipspace.net/2013/11/deutsche-telekom-terastream-designed.html

* Discussion from NFV BoF (Atlanta) etherpad: https://etherpad.openstack.org/p/juno-nfv-bof

* Discussion of the out-of-tree sync mechanism: http://lists.openstack.org/pipermail/openstack-dev/2014-June/037112.html

