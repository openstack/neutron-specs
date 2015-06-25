..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Allow Neutron Port Without IP Address
=====================================

https://blueprints.launchpad.net/neutron/+spec/vm-without-l3-address

Allow to create unaddressed port. i.e. port without l3-address, subnets
and to boot with the port.


Problem Description
===================

Currently VM only with L2 address without ipv4/ip6 address can't be created.
In fact, it is already possible to create a port without IPv4 address,
or without IPv6 address. This means that the current implementation of neutron
port creating could accept empty subnet in request(you will not be forced to
specify the subnet), of cause the VIF type of the created port here is unbound.

Neutron and nova create interfaces with the assumption that the
interface's L2 and L3 assigned addresses are intrinsic attributes;
that an L3 address is not optional, and that traffic should never
be seen by that machine unless it is addressed to the recognised
addresses.

Network applications (for example, routers) often forward traffic that
is not intended for them, and may actually have

 - interface without a primary L3 address, which may be receiving
   traffic for so many disparate addresses that configuring them
   all in Neutron itself is a pointless burden

A typical use case is when a user wishes to deploy a VM which accepts
traffic that is neither IPv4 nor IPv6 in nature, one that accepts is a
superset of v4 and v6 traffic, or one that accepts traffic for a very
wide address range (for either forwarding or termination) and where
the port has no primary address.  In such cases, the VM is not a
conventional application VM.

NOTE: many sentence are shamelessly stolen from [nova-l2-net-without-subnet]_

And we must also note that some L2 driver like l2-pop maybe have problem when
deal with this kind of port because it use arp proxy to answer arp from known
ip address.
And also some service like novnc service may be not work for the port without
IP address.

Proposed Change
===============

Allow to boot VM with port without l3-address.
Actually the current neutron allows to create a port without subnet.
New typical work flow would be as follows (which doesn't work currently)

1. Create neutron L2 network, but any subnets aren't associated to it
2. Boot VM on the network

Or

1. Create neutron L2 network. subnets may or may not be associated to it
2. Create neutron port on the network without fixed ips
3. Boot VM with the created port

In the neutron side, if this kind of port created, security-groups should be
removed and filter like anti-mac-spoofing should be disabled.
And also, if necessary, fix L2/L3 agent codes which depend on that port
should has a fixed ip.

In the nova side, fixed ips and subnet checking exception check like
PortRequiresFixedIP and NetworkRequiresSubnet should be removed carefully.


Data Model Impact
-----------------

None


REST API Impact
---------------

None. Because the current neutron API implementation allows to create a port
without specifying the subnet or any fixed ips in request. So we don't need
a new flag to define it.


Security Impact
---------------

Of cause, unaddressed ports are dangerous to the unwary. But because the
operation of this kind of ports just follow the existing process, only
the network owners and administrators have the privilege to operate the port.
The security impact is minimal.


Notifications Impact
--------------------

L2/L3 agent might be confused without fixed ip address since such a code
path isn't tested.


Other End User Impact
---------------------

None


Performance Impact
------------------

None


IPv6 Impact
-----------

None


Other Deployer Impact
---------------------

None


Developer Impact
----------------

None


Community Impact
----------------

None


Alternatives
------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Yalei Wang
  Zang Rui
  Isaku Yamahata(yamahata)

Other contributors:
  to be added


Work Items
----------
* python neutron client to specify that no fixed ip address is associated
* python nova client to specify that no fixed ip address is associated
* nova neutronv2 network driver, remove the current verification
  PortRequiresFixedIP and NetworkRequiresSubnet, fix bug
  [nova-l2-net-without-subnet]_
* remove the security group or other packages filter of the unaddressed port in
  neutron.
* add tests
* if necessary, fix neutron components.
  especially L2/L3 agents, security group driver


Dependencies
============

Nova neutronv2 network driver would need modification.


Testing
=======

Necessary api/functional tests will be added.


Tempest Tests
-------------

* create port without fixed ip address
  ** connection tests between ports
* boot VM with such ports
* attach/detach such ports to VMs


Functional Tests
----------------

* create port without fixed ip address and tests connectivity between ports


API Tests
---------

None


Documentation Impact
====================

The related part will be updated.


User Documentation
------------------

* nova boot
* neutron port creation


Developer Documentation
-----------------------

None


References
==========

.. [nfv-unaddressed-interface] NFV unaddressed interfaces
  https://review.openstack.org/#/c/97715/

.. [nova-l2-net-without-subnet]
  Creating Neutron L2 networks (without subnets) doesn't work as expected
  https://bugs.launchpad.net/nova/+bug/1039665

.. Make libvirt use the new network model datastructures
  https://review.openstack.org/#/c/11923/
