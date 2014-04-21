=======================================
ML2 Mechanism Driver for the Cisco APIC
=======================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/ml2-cisco-apic-mechanism-driver

This blueprint is to introduce the Cisco APIC to OpenStack
Neutron. The plugin is implemented as an ML2 mechanism driver.


Problem description
===================

The APIC (Application Policy Infrastructure Controller) together with
Cisco Nexus 9000 switches provides programmable, policy-driven network
control.

The mechanism driver proposed here will interact with the APIC to
dynamically manage networking for OpenStack instances. The APIC will
intelligently configure the hardware layer and manage VXLAN overlays
for networks. VXLAN gateways will be implemented by the controller at
the top of rack layer to reduce encapsulation latency in the software
switch layer.

Use Cases:

 1. Application centric policy based networking
 2. External controller managed network fabric
 3. VXLAN Overlay networks



Proposed change
===============

The diagram below provides a high level overview of how the APIC
plugin will integrate into a working environment consisting of
multiple hosts with virtual switches connected by a fabric of Nexus
9000 switches. The APIC Mechanism Driver for ML2 communicated with the
APIC via its REST API.

Flows::

          +–––––––––––––––––––––––––+
          |                         |
          | Neutron Server          |
          | with ML2 Plugin         |
          |                         |
          |          +–––––––––––+  |
  +–––––––+          |   APIC    |  |
  |       |          | Mechanism |  |                  +–––––––––––––––––+
  |       |          |  Driver   |  |                  |                 |
  |  +––––+       +––+–––––––––––+––+      REST API    |   Cisco         |
  |  |    |       |   APIC Client   +––––––––––––––––––+   APIC          |
  |  |    +–––––––+–––––––––––––––––+                  |                 |
  |  |                                                 |                 |
  |  |                                                 +–––+–––+–––––––––+
  |  |    +–––––––––––+––––––––––––––+                     |   |
  |  +––––+ L2 Agent  | Open vSwitch +–––––+               |   |
  |       +–––––––––––+––––––––––––––+     |               |   |
  |       |                          |     |               |   |
  |       |        HOST 1            |     |               |   |
  |       |                          |     |      +––––––––+–––|––––––+
  |       +––––––––––––––––––––––––––+     |      |            |      |
  |                                        +––––––+  +–––––––––+––––––+––+
  |                                               |  |                   |
  |       +––––––––––+–––––––––––––––+            |  |     Cisco         |
  +–––––––+ L2 Agent | Open vSwitch  +––––––––––––+  |   Nexus 9000      |
          +––––––––––+–––––––––––––––+            |  |    Switches       |
          |                          |            +––+                   |
          |        HOST 2            |               |                   |
          |                          |               +–––––––––––––––––––+
          +––––––––––––––––––––––––––+

The APIC mechanism driver updates the APIC with port, network and
subnet changes from Neutron. The APIC configures the physical switch
fabric.

The APIC mechanism driver is designed to operate together with the OVS
mechanism driver for handling network operations and port binding on
the compute nodes.

The APIC mechanism driver implements the following Neutron events:

* Port create/update for compute instances. (Note: the driver does not
  handle port binding -- that is left to the OVS driver.)
* Network create/delete.
* Subnet create.


Alternatives
------------

An alternative solution would be to develop a monolithic plugin. The
biggest advantage of using the ML2 mechanism driver approach is that
it allows us to easily use the existing OVS agent for virtual
switching.

Data model impact
-----------------

Three new models are created by this driver. These models are specific
to this driver and are used for keeping in sync with the APIC.

 1. NetworkEPG: Tracks network to End Point Group mapping in the APIC.

 2. PortProfile: Tracks hardware switch/module/port configuration.

 3. TenantContract: Tracks contracts and filters created on the APIC.

A database migration is included to create the tables for these
models.

No existing models are changed.

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

The performance of ML2 when configured with the APIC driver will be
dependent on the performance of the link between Neutron and the APIC,
and on the responsiveness of the APIC itself.

Other deployer impact
---------------------

The deployer must configure the installation to use the APIC with the
following configuration variables:

* IP address and port number of the APIC.
* Username and password to login as administrator to the APIC.
* VLAN namespace and ranges to be used for OpenStack.
* Node, entity and function profiles describing the switch fabric
  infrastructure.
* Switch-to-host mappings for the compute nodes.

Additionally, the deployer must configure the ML2 plugin to include
the openvswitch mechanism driver before the APIC mechanism driver:

::

  [ml2]
  mechanism_drivers = openvswitch,cisco_apic

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Arvind Somya <asomya>

Henry Gessau <gessau>

Work Items
----------

The work is split up into two parts:

1. APIC Client, models, configuration, unit tests.

   * The APIC Client is a REST Client to the APIC. It maps requests
     to the "Managed Object" model of the APIC. Only the subset of
     the full APIC model needed by the driver is provided.

2. APIC mechanism driver and supporting code.

   * The mechanism driver implements the port, network and subnet
     methods that affect the APIC. It maps Neutron operations into
     operations for the APIC.
   * Supporting code consists of an APIC manager that ensures the
     APIC state is in sync with Neutron.


Dependencies
============

There are no new library requirements. The following third party
library is used:

* requests


Testing
=======

Complete unit test coverage of the code is included.

For tempest test coverage, third party testing is provided. The Cisco
CI reports on all changes affecting this driver. The testing is run in
a setup with an OpenStack deployment (devstack) connected to a live
APIC and a Cisco Nexus 9000 physical switch.


Documentation Impact
====================

Configuration details.


References
==========

http://www.cisco.com/go/apic
