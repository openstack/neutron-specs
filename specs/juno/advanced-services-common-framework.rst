..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================================
Advanced Network Services Framework for Insertion, Steering, and Chaining
=========================================================================

URL of the launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/neutron-services-insertion-chaining-steering

This is an umbrella blueprint specification to capture the high level design
for this framework. Multiple dependent blueprint specifications are being
worked on that will provide the granular details on each component in this
framework. This document will not go into the those details. The design and
plan proposed here is based on the discussions in the advanced services weekly
IRC meetings [1], discussions in the mailing list, and during the Havana and
Icehouse design summit sessions.

Problem description
===================

Neutron, as of the Icehouse release, supports three advanced network services -
Firewall (FWaaS), Loadbalancer (LBaaS), and VPN (VPNaaS), in addition to the
L3 functionality which has also been factored into a pluggable advanced service
implementation. Each of these advanced services is either inserted implicitly
into the tenant topology or explicitly associated with a neutron router.
However, there is no generic service insertion mechanism to facilitate L3, L2,
bump-in-the-wire, and tap insertion modes. Moreover, with more than one
service, it becomes relevant to explore the model of how multiple services can
be sequenced in a chain.

An example, in the context of today’s reference implementations, is the
insertion and chaining of Firewall and VPN services. Each of these reference
implementations rely on the use of IPTables chains to program the relevant
filters and policies to achieve their respective functions. However, in the
absence of a chaining abstraction to express the sequence of these services,
these implementations act independently and the resulting order of operations
is incidental and cannot be controlled.

The goal of this Advanced Services Common Framework is to support the
publishing, instantiation, insertion, and chaining of network services in
Neutron.

Proposed change
===============

The framework needs to support and reconcile the following dimensions:

* Different service:

  - types (e.g. LB, FW, VPN),

  - capabilities of a service type (e.g. transparent FW versus edge FW),

  - levels of a service type (e.g. number of policy rules supported on the FW,
    and perhaps captured in abstract as gold, service, bronze, etc service
    levels)

* Different service manifestations:

  - hardware appliance

  - virtual (VM) appliance

  - host operating system configuration (e.g. IPTables)

  - an independent OS process (e.g. HAProxy)

* Single service insertion

* Chaining of services (linear and graphs)

* Backends supporting combination of services in the same physical or virtual
  appliance

* Requirements of higher level declarative abstractions like Group Policy

* Feasible open source reference implementation

* Backward compatibility (perhaps not as relevant since FWaaS and VPNaaS are
  still experimental, and LBaaS is going through a major redesign)

The framework consists of the following building blocks:

**Flavors framework [2]**

The Flavors framework allows the operator to publish the available service
types and their capabilities. It hides the provider details of the service from
its user.

**Service definition [3]**

Each Neutron service needs to implement a common base service definition. This
defines operations that support the life cycle management of a service instance
of that service and provide for:

* setting the flavor of the service instance

* inserting the service in the data path

* discovering the service

**Service insertion [4]**

Allows the user to request that a particular service be available at a certain
point in the user’s logical topology. Once inserted, the service becomes part
of the data path defined by the regular L2/L3 behavior. Example: a Neutron
router is inserted by creating an interface on a subnet.

**Traffic steering [5]**

A lower level imperative abstraction which allows specifying that traffic be
steered between an ordered list of Neutron ports. The traffic can be further
subject to L2/L3 classification criteria. This abstraction can be leveraged to
realize service chaining, and is particularly helpful in chaining services that
are not known to Neutron and only manifest as Neutron ports (e.g. a user
managed service VM).

**Service Chaining [6]**

This is a declarative abstraction that allows specifying an ordered list/graph
of Neutron services types. The underlying implementation for that chain
performs insertion of each service in the chain, and the steering of traffic
between the services in the specified order. The implementation may choose to
leverage the traffic steering primitives mentioned earlier.

Alternatives
------------

There is no framework for supporting different service insertion modes, for
traffic steering, and/or service chaining.

Data model impact
-----------------

This will be captured in each dependent blueprint.

REST API impact
---------------

This will be captured in each dependent blueprint.

Security impact
---------------

This will be captured in each dependent blueprint.

Notifications impact
--------------------

This will be captured in each dependent blueprint.

Other end user impact
---------------------

This will be captured in each dependent blueprint.

Performance Impact
------------------

This will be captured in each dependent blueprint.

Other deployer impact
---------------------

This will be captured in each dependent blueprint.

Developer impact
----------------

This will be captured in each dependent blueprint.


Implementation
==============

Assignee(s)
-----------

The following people are working on several different aspects of the proposed
framework:

  Sumit Naiksatam (snaiksat)

  Eugene Nikanorov (enikanorov)

  Nachi Ueno (Nachi Ueno)

  Stephen Wong (s3wong)

  Kanzhe Jiang (kanzhe-jiang)

  Mandeep Dhami (mandeep-dhami)

  Carlos Goncalves (cgoncalves)

  João Soares (joaosoares)

  Swaminathan Vasudevan (swaminathan-vasudevan)

  Mohammad Banikazemi (banix)

  Robert Kukura (rkukura)

  Kevin Benton (kevinbenton)

  Gary Duan (gduan)

  Sridar Kandaswamy (skandasw)

  Rajesh Mohan (rajesh.mohan)

  Paul Michali (pmichali)

  Prasad Vellanki (prasad-vellanki)

  Hemanth Ravi (hemanth-ravi)

  Edgar Magana (emagana)

Work Items
----------

This will be captured in each dependent blueprint.

Dependencies
============

This will be captured in each dependent blueprint.

Testing
=======

This will be captured in each dependent blueprint.

Documentation Impact
====================

This will be captured in each dependent blueprint, overall owner: snaiksat

References
==========

[1] https://wiki.openstack.org/wiki/Meetings/AdvancedServices

[2] Flavors framework: https://review.openstack.org/#/c/90070

(Note that the content in the google docs linked below should not be considered
as the final definition or design; those will be presented in the dependent
blueprint specs for review. They are referenced here to provide information on
the discussion till date.)

[3] Service  defintions:
https://docs.google.com/document/d/1fmCWpCxAN4g5txmCJVmBDt02GYew2kvyRsh0Wl3YF2U/edit

[4] Service Insertion:
https://docs.google.com/document/d/1AlEockwk0Ir267U9uFDc-Q6vYsWiAcAoKtCJM0Jc5UI/edit#

[5] Traffic Steering: https://review.openstack.org/#/c/92477

[6] Service Chaining:
https://docs.google.com/document/d/1fmCWpCxAN4g5txmCJVmBDt02GYew2kvyRsh0Wl3YF2U/edit
