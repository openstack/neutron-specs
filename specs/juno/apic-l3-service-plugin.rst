=============================================
Layer 3 service plugin for the Cisco APIC
=============================================

https://blueprints.launchpad.net/neutron/+spec/cisco-apic-l3

This blueprint is to implement Layer 3 features in
the network fabric via a L3 router service plugin
using the Cisco APIC.

Flows

.. blockdiag::

  blockdiag l2_apic {
    Neutron -> APIC -> Nexus_9k;
    Nexus_9k -> Compute1;
    Nexus_9k -> Compute2;
  }

Problem description
===================

The APIC (Application Policy Infrastructure Controller) together with
Cisco Nexus 9000 switches provides programmable, policy-driven network
control.

The Service plugin proposed will interact with the APIC to dynamically
configure Layer 3 inter and intra tenant communications in the network fabric.


Proposed change
===============

This proposal is to introduce a new Layer 3 service plugin that communicates
with the APIC.

The driver implements the following neutron events:
* Add a new router interface
* Delete a router interface

The plugin will implement Layer 3 communications using a construct called
a contract that provides communications in the fabric between various
end point groups (Neutron networks)

Events that trigger the creation of end point groups and subnets (i.e.
create_network and create_subnet) will be handled by the APIC ML2 mechanism
driver and id’s will be stored in a database accessed by a common client and
manager class.

Due to hardware limitations as of this writing this service plugin will
only handle add/remove router_interface (internal gateway) events. The service
plugin may be expanded in the future to also handle all neutron L3 events.

Alternatives
------------
The alternative approach is to use the open source agent based layer 3 router
plugin. The agent based approach does not implement any policies that are
centric to the APIC’s ACI (Application Centric Infrastructure) based design.

Data model impact
-----------------
n.a.

REST API impact
---------------
n.a

Security impact
---------------
n.a

Notifications impact
--------------------
n.a.

Other end user impact
---------------------
n.a.

Performance Impact
------------------
The service plugin is triggered instead of polled, there are no
changes to any existing code patterns. The potential bottleneck
for this plugin would be the link between neutron and the APIC.

Other deployer impact
---------------------
There are no config options specific to the Layer 3 plugin, it relies
on the configuration options of the ML2 apic mechanism driver.

Developer impact
----------------
n.a.


Implementation
==============

Assignee(s)
-----------

Arvind Somya <asomya>

Work Items
----------
Single work item for the L3 APIC service plugin.


Dependencies
============
Depends on the APIC ML2 blueprint:
https://blueprints.launchpad.net/neutron/+spec/ml2-cisco-apic-mechanism-driver


Testing
=======
Complete unit test coverage of the code is included.

For tempest test coverage, third party testing is provided. The Cisco
CI reports on all changes affecting this driver. The testing is run in
a setup with an OpenStack deployment (devstack) connected to a live
APIC and a Cisco Nexus 9000 physical switch.


Documentation Impact
====================
Deployment documentation on how to configure and deploy this service plugin
will be documented in the Openstack wiki.


References
==========
http://www.cisco.com/go/apic
