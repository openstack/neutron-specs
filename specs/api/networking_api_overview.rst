============================
Networking API v2.0 Overview
============================

The Neutron project provides virtual networking services among devices
that are managed by the
`OpenStack <http://wiki.openstack.org/OpenStack>`__ compute service.

The Networking API v2.0 combines the `Quantum API
v1.1 <http://docs.openstack.org/api/openstack-network/1.0/content/>`__
with some essential Internet Protocol Address Management (IPAM)
capabilities from the `Melange
API <http://melange.readthedocs.org/en/latest/apidoc.html>`__.

These IPAM capabilities enable you to:

-  Associate IP address blocks and other network configuration settings
   required by a network device, such as a default gateway and
   dns-servers settings, with an OpenStack Networking network.

-  Allocate an IP address from a block and associate it with a device
   that is attached to the network through an OpenStack Networking port.

To do this, the Networking API v2.0 introduces the subnet entity. A
subnet can represent either an IP version 4 or version 6 address block.
Each OpenStack Networking network commonly has one or more subnets. When
you create a port on the network, an available fixed IP address is
allocated to it from one the designated subnets for each IP version.
When you delete the port, the allocated addresses return to the pool of
available IPs on the subnet. Networking API v2.0 users can choose a
specific IP address from the block or let OpenStack Networking choose
the first available IP address.

Note
~~~~

The Quantum API v1.\ ``x`` was removed from the source code tree. To use
the Quantum API v1.\ ``x``, install the Quantum Essex release.


Concepts
--------

Use the Networking API v2.0 to manage the following entities:

-  **Network**. An isolated virtual layer-2 domain. A network can also
   be a virtual, or logical, switch.

-  **Subnet**. An IP version 4 or version 6 address block from which IP
   addresses that are assigned to VMs on a specified network are
   selected.

-  **Port**. A virtual, or logical, switch port on a specified network.

These entities have auto-generated unique identifiers and support basic
create, read, update, and delete (CRUD) functions with the **POST**,
**GET**, **PUT**, and **DELETE** verbs.

Network
~~~~~~~

A network is an isolated virtual layer-2 broadcast domain that is
typically reserved for the tenant who created it unless you configure
the network to be shared. Tenants can create multiple networks until the
thresholds per-tenant quota is reached.

In the Networking API v2.0, the network is the main entity. Ports and
subnets are always associated with a network.

The following table describes the attributes for network objects:

Subnet
~~~~~~

A subnet represents an IP address block that can be used to assign IP
addresses to virtual instances. Each subnet must have a CIDR and must be
associated with a network. IPs can be either selected from the whole
subnet CIDR or from allocation pools that can be specified by the user.

A subnet can also optionally have a gateway, a list of dns name servers,
and host routes. This information is pushed to instances whose
interfaces are associated with the subnet.



Port
~~~~

A port represents a virtual switch port on a logical network switch.
Virtual instances attach their interfaces into ports. The logical port
also defines the MAC address and the IP address(es) to be assigned to
the interfaces plugged into them. When IP addresses are associated to a
port, this also implies the port is associated with a subnet, as the IP
address was taken from the allocation pool for a specific subnet.

**Table Port Attributes**


High-level task flow
--------------------

The high-level task flow for OpenStack Networking involves creating a
network, associating a subnet with that network, and booting a VM that
is attached to the network. Clean-up includes deleting the VM, deleting
any ports associated with the network, and deleting the networks.
OpenStack Networking deletes any subnets associated with the deleted
network.

**To use OpenStack Networking - high-level task flow**

#. **Create a network**

   The tenant creates a network.

   For example, the tenant creates the ``net1`` network. Its ID is
   net1\_id.

#. **Associate a subnet with the network**

   The tenant associates a subnet with that network.

   For example, the tenant associates the ``10.0.0.0/24`` subnet with
   the ``net1`` network.

#. **Boot a VM and attach it to the network**

   The tenant boots a virtual machine (VM) and specifies a single NIC
   that connects to the network.

   The following examples use the nova client to boot a VM.

   In the first example, Nova contacts OpenStack Networking to create
   the NIC and attach it to the ``net1`` network, with the ID net1\_id:

   .. code::

       $ nova boot <server_name> --image <image> --flavor <flavor> --nic net-id=<net1_id>

   In a second example, you first create the ``port1``, port and then
   you boot the VM with a specified port. OpenStack Networking creates a
   NIC and attaches it to the ``port1`` port, with the ID port1\_id:

   .. code::

       $ nova boot <server_name> --image <image> --flavor <flavor> --nic port-id=<port1_id>

   OpenStack Networking chooses and assigns an IP address to the
   ``port1`` port.

   For more information about the **nova boot** command, enter:

   .. code::

       $ nova help boot

#. **Delete the VM**

   The tenant deletes the VM.

   Nova contacts OpenStack Networking and deletes the ``port1`` port.

   The allocated IP address is returned to the pool of available IP
   addresses.

#. **Delete any ports**

   If the tenant created any ports and associated them with the network,
   the tenant deletes the ports.

#. **Delete the network**

   The tenant deletes the network. This operation deletes an OpenStack
   Networking network and its associated subnets provided that no port
   is currently configured on the network.

Plug-ins
--------

Virtual networking services are implemented through a plug-in. A plug-in
can use different techniques and technologies to provide isolated
virtual networks to tenants. A plug-in also provides other services,
such as IP address management. Because each plug-in implements all the
operations included in Networking API v2.0, do not be concerned about
which plug-in is used.

However, some plug-ins might expose additional capabilities through API
extensions, which this document discusses. For more information about
the extensions exposed by a particular plug-in, see the plug-in
documentation.

