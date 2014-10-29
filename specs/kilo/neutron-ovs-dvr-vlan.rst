..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Support for VLAN networks in Distributed Virtual Router (DVR)
=============================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/neutron-ovs-dvr-vlan

With Juno deployment, users of openstack neutron can deploy distributed virtual
routers as referred in [1].

DVRs enable routing between tenant VMs without the need to deploy a
centralized node to host tenant routers. Tenants can create distributed
routers and such routers are made available on-demand basis on the compute
servers that have tenant VMs on subnets hosted by the router.

However, in Juno release of neutron, distributed router interfaces that are
managed can belong only to VXLAN (or) GRE network types.

The capability to host VLAN network types as distributed router interfaces was
not available.

Problem Description
===================

As explained in the introduction section above, Juno release of openstack
makes available distributed virtual routers.  However, the implementation
available is functional only for network_types for VXLAN and GRE.

This blueprint will address the following gaps:

1. Enables VLAN Networks to be hosted as distributed router interfaces
More specifically, if two networks are attached as interfaces of a dvr,
of which both the networks are VLAN type, then VMs on such network can
route traffic between each other transparent to the tenant, via
this blueprint.


2. Enables a distributed router to route packets between a VLAN network,
VXLAN network and GRE network.
More specifically, if two networks are attached as interfaces of a dvr,
of which one is a VLAN network and another is a VXLAN network, then VMs
on such network can route traffic between each other transparent to the
tenant, via this blueprint.
Similarly, this blueprint also enables routed communication between
one interface of the dvr being on VLAN network and another interface
being on GRE network.
Also, this blueprint also enables routed communication between one
interface of a dvr being on a VXLAN network and another interface
being on a GRE network.

We are intending to clarify here that this blueprint would enable only routed
traffic to pass through disparate network types (only VLAN, VXLAN, GRE).
This blueprint does not propose a gateway to switch traffic across disparate
networks.

Proposed Change
===============

We will be making changes to the following components of the Openstack
Neutron as part of implementation of this blueprint:

a. The DVR RPCs serviced by the ML2 Plugin will be enhanced to provide
physical network information that will be used by the OVS Neutron
Agent to plumb VLAN network-type ports as DVR interfaces.

b. The OVS DVR Neutron Agent will be enhanced to embrace VLAN network-type
thereby allowing tenant VMs that are on VLAN networks to communicate via
distributed router.   Also it will be enhanced to enable tenant VMs (one
on VLAN and other on VXLAN/GRE network) to communicate via
distributed router.
By enhancement, we want to refer that OVS Rules will be added to
integration-bridge and physical-bridges by the OVS DVR Neutron Agent
in order to enable routing between VMs on different VLAN networks and
also to enable routing between VMs that each reside on a VLAN network
and a VXLAN/GRE network.

So, the blueprint will enable DVR openvswitch implementation to embrace
VLAN network-types.

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

IPv6 Impact
-----------

This change will not have impact on IPv6 in Neutron.

The Distributed Virtual Router (dvr) in [1] does not support IPv6
networks.  And, so the DVR feature must first be enhanced to support
IPv6 as a separate blueprint.  Until then IPv6 tenant networks
will not able to take advantage of the DVR feature in neutron.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Community Impact
----------------

The Neutron community embraced the DVR feature in the intent that
it resolves parity issues with nova-network.   This blueprint will
further close gaps to bring-in more parity.

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

The author of this blueprint will be posting code for reviews
by the community.

Primary assignee:
  vivekanandan.narasimhan@hp.com

Other contributors:
  blak111@gmail.com (kevin benton)
  swaminathan.vasudevan@hp.com (review)
  rajeev.grover@hp.com (review)
  michael.smith6@hp.com (review)

Work Items
----------

The following are the work items involved for execution of this
blueprint:

1. Primary code-piece development in Neutron to enhance existing DVR logic
to embrace VLAN networks (in addition to retaining the functionality to host
VXLAN/GRE networks).

2. Unit test cases development to ensure test coverage for this feature.

3. There will be functional test cases that will be added in initial commits
to test the feature.

4. In the longer term (in next release), the existing DVR CI would be enhanced to
embrace VLAN-type networks as DVR interfaces.

Dependencies
============

Depends on the Distributed Virtual Router blueprint as in [1].

Testing
=======

As mentioned in Work Items, initially functional tests will be made
available to test this feature.  Later the existing DVR CI (available at gate)
will be enhanced to embrace testing VLAN networks with DVR.

Tempest Tests
-------------

None

Functional Tests
----------------

Functional tests cases would be provided to test this feature.

API Tests
---------

None

Documentation Impact
====================

There will be documentation impact as we need to include in the openstack
neutron manual, that DVR is capable of supporting VLAN network-type.
The auther and the extended existing DVR team will work with
neutron documentation in including this feature.

User Documentation
------------------

Openstack Networking Administration Guide will be updated to reflect
the capability brought in by this blueprint into DVR.

Developer Documentation
-----------------------

None

References
==========

[1] DVR blueprint: http://specs.openstack.org/openstack/neutron-specs/specs/juno/neutron-ovs-dvr.html

