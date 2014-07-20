..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
ML2 Mechanism Driver for Cisco UCS Manager
==========================================

URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/ml2-ucs-manager-mechanism-driver

The purpose of this blueprint is to add support for Cisco UCS Manager into
Openstack Neutron. This plugin for Cisco UCS Manager is implemented as a ML2
mechanism driver.

Problem description
===================

This section includes a brief introduction to the SR-IOV and Cisco VM-FEX
technologies in addition to a detailed description of the problem:

1. This mechanism driver needs to be able to configure Cisco VM-FEX on specific
Cisco NICs on the UCS by communicating with a UCS Manager.

2. Cisco VM-FEX technology is based on SR-IOV technology which allows a single
PCIe physical device (PF - physical function) to be divided into multiple
logical devices (VF - virtual functions). For more details on Cisco VM-Fex
technology, please refer to: http://www.cisco.com/c/en/us/solutions/data-center-virtualization/data-center-virtual-machine-fabric-extender-vm-fex/index.html

3. With SR-IOV and Cisco VM-FEX a VM's port can be configured in either the
"direct" or "macvtap" modes. In the "direct" mode, the VM's port is connected
directly to the VF and to a macvtap device on the host in the "macvtap" mode.
In both these modes, the VM's traffic completely bypasses the hypervisor,
sending and receiving traffic directly to and from the vNIC and thus the
upstream switch. This results in a significant increase in throughput on the VM
and frees up CPU resources on the host OS to handle more VMs. Due to this
direct connection with the upstream switch, the "direct" mode does not support
live migration of the VMs that it is attached to.

4. Cisco VM-FEX technology is based on the 802.1qbh and works on top of the
SR-IOV technology using the concept of port profiles. Port profiles are
configuration entities that specify additional config that needs to be applied
on the VF. This config includes the vlan-id, QoS (not applicable in Openstack
for now) and the mode (direct/macvtap).

5. This mechanism driver needs to configure port profiles on the UCS Manager
and pass this port profile to Nova so that it can stick it into the VMs domain
XML file.

6. This mechanism driver is responsible for creating, updating and deleting
port profiles in the UCS Manager and maintaining the same info in a local DB.

7. This mechanism driver also needs to support SR-IOV capable Intel NICs on the
UCS servers.

8. Security groups cannot be applied on SR-IOV and VM-FEX ports. Further work
needs to be done to handle security groups gracefully for these ports. It will
be taken up in a different BP in the next iteration. (Since the VFs appear as
interfaces on the upstream switch, ACLs can be applied on the VFs at the
upstream switch.)

9. Multi-segment networks are also currently not supported by this BP. It will
be added as part of a different BP in the next iteration.


Proposed change
===============

1. This ML2 mechanism driver communicates to the UCS manager via UCS Python SDK.

2. There is no need for a L2 agent to be running on the compute host to
configure the SR-IOV and VM-FEX ports.

3. The ML2 mechanism driver takes care of binding the port if the pci vendor
info in the binding:profile portbinding attribute matches the pci device
attributes of the devices it can handle.

4. The mechanism driver also expects to get the physical network information
as part of port binding:profile at the time of bind port.

5. When a neutron port is being bound to a VM, the mechanism driver
uses the segmentation-id associated with the network to determine if a new
port profile needs to be created. According to the DB maintained by the ML2
driver, if a port profile with that vlan_id already exists, then it re-uses
this port profile for the neutron port being created.

6. If the ML2 driver determines that an existing port profile cannot be re-used
it tries to create a new port profile on the UCS manager using the vlan_id
from the network. Since port profile is a vendor specific entity, we did not
want to expose this to the cloud admin or the tenant. So, port profiles are
created and maintained completely behind the scenes by the ML2 driver.

7. Port profiles created by this mechanism driver will have the name
"OS-PP-<vlan-id>".
The process of creating a port profile on the UCS Manager involves:
A. Connecting to the UCS manager and starting a new transaction
B. Creating a Fabric Vlan managed object corresponding to the vlan-id
C. Creating a vNIC Port Profile managed object that is associated with
the above Fabric Vlan managed object.
D. Creating a Profile Client managed object that corresponds to the vNIC
Port Profile managed object.
E. Ending the current transaction and disconnecting from UCS maanager

8. Once the above entities are created on the UCS manager, the ML2 driver
populates the profile:vif_details portbindings attribute with the profile_id
(name of the port profile). Nova then uses Neutron V2 API to grab the
profile_id and populates the VM's domain XML. After the VM is successfully
launched by libvirt, the configuration of VM-FEX is complete.

9. In the case of NICs that support SR-IOV and not VM-FEX (for example, the
Intel NIC), the portbinding profile:vif_details attribute is populated with
the vlan_id. This vlan_id is then written into the VM's domain XML file by
Nova's generic vif driver.


Alternatives
------------
None.

Data model impact
-----------------

One new data model is created by this driver to keep track of port profiles
created by Openstack. Other port profiles can exist on the UCS Manager that this
driver does not care about.

PortProfile: Tracks the vlan-id of the port associated with a given port
profile.

__tablename__ = 'ml2_ucsm_port_profiles'

profile_id = sa.Column(sa.String(64), nullable=False, primary_key=True)
vlan_id = sa.Colum(sa.Integer(), nullable=False)

The profile_id to port_id mapping is kept track of via ml2_port_bindings table
where the profile_id is stored in vif_details.

REST API impact
---------------
None.

Security impact
---------------
The connection to the XML API layer on the UCS Manager is via HTTP/HTTPS.

Traffic from the VM completely bypasses the host, so no security groups are
enforced when this mechanism driver binds the port.

Notifications impact
--------------------
None.

Other end user impact
---------------------
No user impact. As mentioned earlier, did not want to expose the port profile_id
to the user since this is a vendor specific entity. Instead, managing
allocation of port profiles internally within the driver.

Performance Impact
------------------
The ML2 driver code would have to conditionally communicate with the UCS Manager
to configure, update or delete port profiles and associated configuration. These
tasks would have a performance impact on Neutron's responsiveness to a command
that affects port config.

Other deployer impact
---------------------
The deployer must provide the following in order to be able to connect to a UCS
Manager and add support for SR-IOV ports.

1. IP address of UCS Manager.
2. Admin Username and password to log into UCS Manager.
3. Add "cisco_ucsm" as a ML2 driver to handle SR-IOV port configuration.
4. Add the "vlan" type driver. Currently, this mechansim driver supports only
   the VLAN type driver.

These should be provided in:
/opt/stack/neutron/etc/neutron/plugins/ml2/ml2_conf_cisco.ini.

Example:
[ml2_cisco_ucsm]

# Hostname for UCS Manager
# ucsm_ip=1.1.1.1

# Username for the UCS Manager
# ucsm_username=username

# Password for APIC controller
# ucsm_password=password

The deployer should also install the Cisco UCS Python SDK for this ML2 mechanism
driver to connect and configure the UCS Manager. The SDK and install
instructions can be found at: https://github.com/CiscoUcs/UcsPythonSDK.

Developer impact
----------------
None.

Implementation
==============

Assignee(s)
-----------

Sandhya Dasu <sadasu>

Work Items
----------
Work Items can be roughly divided into the following tasks:
1. Mechanism driver handles port create, update and delete requests.
2. Network driver handles communication with UCS manager. This communication is
triggered by an operation peformed on a port by the mechanism driver.
3. Unit test cases to test the mechanism driver and network driver code.
4. Tempest test cases to peform end-to-end and functional testing of 1 and 2.


Dependencies
============
1. This mechanism driver depends on third party UCS python SDK (located at :
https://github.com/CiscoUcs/UcsPythonSDK) to communicate with the UCS Manager.

2. For SR-IOV ports to be actually scheduled and assigned to a VM, some non-vendor specific Nova code is required. This effort is tracked via:
https://blueprints.launchpad.net/nova/+spec/pci-passthrough-sriov

Testing
=======

Third party tempest testing will be provided for this mechanism driver.
Cisco CI will start reporting on all changes affecting this driver. The third
party tempest tests will run on a setup which runs Openstack (devstack) code on
a multi-node setup that is connected to a UCS Manager system.

Documentation Impact
====================

Details of configuring this mechanism driver.

References
==========
1. Here is the link to the larger discussion around PCI passthrough ports:
https://wiki.openstack.org/wiki/Meetings/Passthrough

2. Useful links on VM-FEX - http://www.cisco.com/c/en/us/solutions/data-center-virtualization/data-center-virtual-machine-fabric-extender-vm-fex/index.html and
https://www.youtube.com/watch?v=8uCU9ghxJKg
