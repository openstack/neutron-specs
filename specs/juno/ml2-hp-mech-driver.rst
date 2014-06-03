===========================================================
HP ML2 Mechanism Driver for the Configuring HP TOR switches
===========================================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/vxlan-gateway

This blueprint is to introduce HP mechanism driver for configuring
HP TOR switches for VXLAN gateway support.


Problem description
===================

This blueprint specifies need of HP’s modular L2 mechanism driver to automate 
the provisioning of HP’s ToR switches using OVSDB.In current Openstack neutron 
deployment VM’s attached to the overlay network could not communicate to the network 
devices/bare-metal servers attached to the physical networks.  We need a driver in 
ML2 to synchronize the overlay information, Hypervisor VTEP, VM MAC etc., to OVSDB 
and HP’s ToR will read the OVSDB to provide access to bare-metal (physical) 
appliances to the logical network.

Use Cases:

 1. VMs in VXLAN Overlay networks talks to physical server that is connected to 
    the HP TOR switches
 2. Auto synchronize the neutron DB(VNI , L2 Network details etc) to the OVSDB 
 3. Software VTEP information to the OVSDB
 4. Poll OVSDB for physical servers , physical port added/deleted details and update in neutron DB



Proposed change
===============

The diagram below provides a high level overview of how the HP driver interacts with 
OVSDB that is running on remote HOST .


Flows::

          +–––––––––––––––––––––––––+
          |                         |
          | Neutron Server          |
          | with ML2 Plugin         |
          |                         |
          |          +–––––––––––+  |
  +–––––––+          |   HP      |  |
  |       |          | Mechanism |  |                  +–––––––––––––––––+
  |       |          |  Driver   |  |                  |                 |
  |  +----|          |           |  |                  |                 |
  |  |    |          +-----------+  |+–––––––––––––––––|     OVSDB       |
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
  |       +––––––––––+–––––––––––––––+            |  |     HP            |
  +–––––––+ L2 Agent | Open vSwitch  +––––––––––––+  |    5930           |
          +––––––––––+–––––––––––––––+            |  |    Switches       |
          |                          |            +––+                   |
          |        HOST 2            |               |                   |
          |                          |               +–––––––––––––––––––+
          +––––––––––––––––––––––––––+

The HP mechanism driver updates the OVSDB with VNI,VM Mac , SW VTEP information
from the neutron and the OVSDB configures these values in the TOR switch




Alternatives
------------

An alternative solution would be to develop a monolithic plugin. The
biggest advantage of using the ML2 mechanism driver approach is that
it allows us to easily use the existing OVS agent.

Data model impact
-----------------


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

None

Other deployer impact
---------------------

Additionally, the deployer must configure the ML2 plugin to include
the openvswitch mechanism driver before the HP mechanism driver:

::

  [ml2]
  mechanism_drivers = openvswitch,hp_mech

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Selvakumar S(sels)


Work Items
----------

The work is split up into two parts:

1. Leveraging jsonrpc library 

   * This driver uses jsonrpclib (JSON-RPC 1.0) library for communicating 
     to the OVSDB server that is running separately from ToR unlike Arista.

2. Implement additional network entity called NetworkGateway in the driver 
   for handling connecting l2 networks to the physical l2 gateway 


Dependencies
============

There are no new library requirements. The following third party
library is used:

* requests


Testing
=======


The testing is run in a setup with an OpenStack deployment (devstack) connected 
to a HP Vendor controller that configures the TOR switch


Documentation Impact
====================

Configuration details.


References
==========

https://blueprints.launchpad.net/neutron/+spec/l2-gateway

