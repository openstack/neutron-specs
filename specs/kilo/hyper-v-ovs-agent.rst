..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Neutron OVS agent on Windows
==========================================

https://blueprints.launchpad.net/neutron/+spec/hyper-v-ovs-agent

Thanks to the porting of Open vSwitch to Hyper-V is is now also possible to
have the Neutron OVS L2 agent to work on both Linux and Windows. Most of the
porting activity consists in replacing Linux specific dependencies with
portable alternatives in the Neutron OVS agent, considering in particular that
the agent primarily execs OVS CLI commands, and that OVS has an identical
command line interface on both Linux and Windows.

Problem Description
===================

Current implementation of OVS agent has a few GNU/Linux specific components:

    * The use of rootwrap. This is not necessary on Windows.
    * fcntl is used in agent.linux.utils which is imported by
      agent.linux.ovs_lib. It is also used by eventlet to set non blocking I/O.
      fcntl is not available on Windows, so any dependency on this will be
      replaced with Windows compatible alternatives.
    * ovsdb_monitor.SimpleInterfaceMonitor currently only works on GNU/Linux.
      The reason for this is that agent.linux.async_process.AsyncProcess() uses
      platform specific components like the kill command. A Windows alternative
      will be created to address this.

Proposed Change
===============

Considering that OVS agent currently uses a series of exec calls to ovs-vsctl
and ovs-ofctl ovs_lib and parts (if not all) of utils will be almost identical
on both Linux and Windows. This implies that they will have to be moved from
neutron.agent.linux to a common location, e.g.: neutron.agent.common.
A base class will be created and platform specific differences will be
abstracted (for example, the need for rootwrap).

Data Model Impact
-----------------

None

REST API Impact
---------------

None.

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

IPv6 Impact
-----------

None.

Other Deployer Impact
---------------------

This change will require the user to install Open vSwitch on Hyper-V before
deploying neutron ovs agent, and make the binaries
available in the PATH.

Developer Impact
----------------

None.

Community Impact
----------------

None

Alternatives
------------

The Neutron Hyper-V agent offers already support for the ML2 plugin, which in
turn allows interoperability with
othe mechanism drivers, inclusing Open vSwitch.
The main limitation of this option is that since Hyper-V does not support
VXLAN or GRE encapsulation natively, networking is limited to the VLAN and
flat options.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gabriel-samfira


Work Items
----------

* Move neutron.agent.linux.ovs_lib and parts of neutron.agent.linux.utils to
  neutron.agent.common
* Create a base class for interactig with Open vSwitch, and abstract away the
  differences that are platform specific.
* remove the need for roowrap. Only enable it on GNU/Linux
* update/add unit tests accordingly


Dependencies
============

This change does not directly depend on any other blueprint. It is however
related to:

    https://blueprints.launchpad.net/nova/+spec/hyper-v-ovs-vif


Testing
=======

This feature will be tested by the Hyper-V CI.

Tempest Tests
-------------

Current tempest tests are sufficient.

Functional Tests
----------------

Functional tests will be added as needed.

API Tests
---------

None.


Documentation Impact
====================

User Documentation
------------------

Documentation should reflect that OVS agent now works on Hyper-V as well.

Developer Documentation
-----------------------

None.

References
==========

[1] https://blueprints.launchpad.net/nova/+spec/hyper-v-ovs-vif
[2] https://github.com/openvswitch/ovs
[3] http://www.cloudbase.it/open-vswitch-on-hyper-v
[4] https://github.com/gabriel-samfira/neutron/tree/windows-ovs-agent-poc

