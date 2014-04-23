..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
ML2 Mechanism Driver for embedded switch SR-IOV NIC
===================================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/ml2-sriov-nic-switch

This blueprint is to add ML2 Mechanism Driver for SR-IOV capable NIC based
switching (HW VEB), such as Mellanox ConnectX Family.


Problem description
===================

SR-IOV capable NIC, such as Mellanox ConnectX Family cards, provide
embedded switching capability. As part of the both nova and neutron
enchacements to support SR-IOV ports, the mechanism driver proposed
by this blueprinht will initially support VLAN network type.
Nova VIF Driver will take care of setting VLAN ID via libvirt network
interface XML.
The mechanism driver will require L2 agent to make run time networking
setting updates on SR-IOV ports.

Proposed change
===============

The Ml2 mechanism driver will be based on Simple Agent Mechanism Driver.
The Ml2 mechanism driver will be capable to bind ports that match supported
criteria listed below.
It will support DIRECT and MACVTAP vnic types.
It will support PCI VENDOR INFO initially matching Mellanox ConnectX Family.
There will be an option to load the list of supported PCI VENDOR INFO
from configuration file.
The mechanism driver will include vlan_id attribute in port vif_details
dictionary to provide nova vif driver with vlan id details.
The mechanism driver will expect to get port binding:profile populated with
physical_network and pci_vendor_info to identify if should bind the port and
on which physical network to bind.
To support L2 agent rpc calls, _device_to_port_id method of ml2 rpc module
should be modified to service devices that are not prefixed by tap.

Alternatives
------------

An alternative solution would be to develop a monolithic plugin.
Since SR-IOV resources are limited and support less flexible configuration
options as virtual ports, it makes sense to provide SR-IOV ports aside with
virtual ports.

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

Possible impact on performance may be due to pci_vendor_info match check during
the attemt to bind port.

Other deployer impact
---------------------

The deployer must configure the following configuration variables:

* VLAN namespace and ranges to be used for OpenStack.
* Supported pci vendor info records if different from default list.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
Irena Berezovsky <irenab>


Work Items
----------

1. Ml2 Mechanism Driver for NIC based switch
2. Ml2 Agent for NIC based switch
3. ML2 RPC support for SR-IOV devices require by L2 agent

Dependencies
============

Nova vif driver is required to support SR-IOV NIC based virtual interface.
This functionality is added following the nova blueprint:
https://blueprints.launchpad.net/nova/+spec/pci-passthrough-sriov

Testing
=======

The code will be covered with unit tests.
For tempest test coverage, third party testing is provided.
The tests are run in devstack setup on physical server with SR-IOV
ConnectX Family card.

Documentation Impact
====================

Configuration Reference guide will be updated from the code.

References
==========

SR-IOV support discussion details are here:

https://wiki.openstack.org/wiki/Meetings/Passthrough
