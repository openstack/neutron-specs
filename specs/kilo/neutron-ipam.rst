..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Pluggable IPAM Subsystem in Neutron
==========================================

Launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/neutron-ipam

This blueprint introduces the pluggable IPAM subsystem in Neutron that allows
flexible control over the lifecycle of network resources like the following:

* Fixed IP addresses assigned to Neutron ports
* Floating IP addresses
* Network address ranges, i.e. sub-networks

Problem Description
===================

* Users have a requirement to integrate OpenStack into their existing
  infrastructure that uses external IPAM.

* Currently most (if not all) Neutron plugins leverage an IPAM implementation
  that is embedded in the db_base_plugin implementation. While this works well
  in a self-contained system, it makes it difficult or impossible to integrate
  with an external IPAM backend without terrible hacks.

The current architecture of Neutron does not allow users to implement
their own IPAM mechanism in any other way other than introducing their own
core plugin. That is, while the DHCP provider can be changed, the actual
allocation logic cannot.

Proposed Change
===============

Overview
--------

The proposed change is the addition of a well-defined, abstract IPAM interface,
and the refactoring of the existing NeutronDbPluginV2 to utilize that interface
rather than directly perform IPAM actions on its own. A related blueprint [#]_
is proposed to implement a reference IPAM driver that captures the current
behavior.

.. [#] https://blueprints.launchpad.net/neutron/+spec/reference-ipam-driver

IPAM Interface
~~~~~~~~~~~~~~

IPAM subsystem architecture overview is shown below. There is an abstract base
class defined, IPAMDriver. There will be a reference implementation of this
class - NeutronIPAM - that will encapsulate the current behavior. The IPAMDriver
can be subclassed by third parties to implement different IPAM behaviors, such
as different subnet or IP allocation strategies, or access to an external IPAM
system.

The Neutron Plugins will call into the driver for IPAM functionality.

The driver interface will be synchronous.

   .. blockdiag::

    blockdiag admin {
        group {
            fontsize = 20;
            label = "Plugins";
            orientation = portrait;

            NeutronDbPluginV2 -> Ml2Plugin, LinuxBridgePluginV2,
                                                "...PluginV2" [dir = back];
        }
        NeutronDbPluginV2 -> IPAMDriver [ label = "uses" ];
        group {
            fontsize = 20;
            label = "IPAM Drivers";
            orientation = portrait;

            IPAMDriver -> NeutronIPAM  [style = dotted, dir = back,
                                        label = implements, fontsize = 8];
        }
    }

The IPAM implementation will define abstract Request classes for subnets and
addresses. These classes enable the caller to request subnet or IP allocations
using criteria other than the explicit subnet or address, though implementation
of request criteria other than the existing behavior is not within scope of
this blueprint.

See [#]_ for the interface definition. The details of this interface are
not part of this spec, and work on this interface may continue beyond approval
of this spec.

.. [#] https://review.openstack.org/#/c/134339/

Interaction Examples
~~~~~~~~~~~~~~~~~~~~

An example interaction scenario between Neutron Plugins and IPAM are shown in
the following diagram. In this diagram and the one that follows, "Pluggable
IPAM" represents an optional external IPAM system. These diagrams are intended
as examples; the specific details of each flow will be defined during
implementation.

   .. seqdiag::

    seqdiag {
        "Neutron Plugin" -> NeutronDbPluginV2 [label = "create_subnet"];
        NeutronDbPluginV2 -> "IPAM Driver" [label = "allocate_subnet"];
        "IPAM Driver" -> "Pluggable IPAM" [label = "Allocate Subnet"];
        "IPAM Driver" <- "Pluggable IPAM" [label = "subnet"];
        NeutronDbPluginV2 <- "IPAM Driver" [label = "subnet"];
        NeutronDbPluginV2 -> "Neutron DB" [label = "subnet"];
        NeutronDbPluginV2 <- "Neutron DB";
        "Neutron Plugin" <- NeutronDbPluginV2 [label = "subnet"];
    }

Here is another flow demonstrating the call to create a port. In this case,
the IPAM driver is called in order to retrieve a subnet, *or* allocate the
subnet if not found. That is, the "get_subnet" call may simply retrieve an
existing subnet from the Neutron database, or it may go to an external IPAM
system to query or allocate the subnet. The behavior is left to the discretion
of the driver.

The IPAMSubnet object is used for the IP allocation.

   .. seqdiag::

    seqdiag {
        "Neutron Plugin"  ->  NeutronDbPluginV2 [label = "create_port"];
        NeutronDbPluginV2 -> "IPAM Driver" [label = "get_subnet"];
        NeutronDbPluginV2 <- "IPAM Driver" [label = "IPAMSubnet"];
        NeutronDbPluginV2 -> IPAMSubnet [label = "allocate_ip"];
        IPAMSubnet -> "Pluggable IPAM" [label = "Allocate IP"];
        IPAMSubnet <- "Pluggable IPAM" [label = "IP"];
        NeutronDbPluginV2 <- IPAMSubnet [label = "IP"];
        NeutronDbPluginV2 -> "Neutron DB" [label = "port, IP data"];
        NeutronDbPluginV2 <- "Neutron DB";
        "Neutron Plugin" <- NeutronDbPluginV2 [label = "port, IP data"];
    }


Driver Creation
~~~~~~~~~~~~~~~

The IPAMDriver will contain a factory method to generate specific driver
instances. There will be a driver instance per SubnetPool (see [#]_). However,
in this release only a single driver will be supported across the deployment.

.. [#] https://blueprints.launchpad.net/neutron/+spec/subnet-allocation

Note that this BP will not provide any database or API for the SubnetPool. As
part of this BP only a minimal implementation will be created.

The IPAM driver to use for a SubnetPool is specified through the configuration
file /etc/neutron/neutron.conf. The default value will point to the reference
NeutronIPAM driver.

Refactoring
~~~~~~~~~~~

The existing NeutronDbPluginV2 must be refactored to utilize the new IPAM
interface. Several core plugins make calls to IPAM-related private methods in
the NeutronDbPluginV2. Stub versions of those methods must be left in place and
be refactored to utilize the IPAM interface, or the plugins must themselves be
refactored to avoid calling private methods of the base class. The current
implementation within those methods will be moved to the reference driver.

While the driver will enable an external IPAM system to provide the
authoritative response on whether to allocate a new address or subnet, the
Neutron database will still be required to have an accurate representation
of the currently allocated subnets and IP addresses. Queries for existing
allocations will still access the local database rather than call out through
the driver. The synchronization of external IPAM and Neutron during initial
migration and for ongoing verification purposes is the responsibility of the
driver author, either within the driver or external to it.

The DB activities currently done in NeutronDbPluginV2 would better be handled
via composition rather inheritance. The base plugin could have a database
handler object that performs these functions. This would enable the database
transaction to be performed outside (after) the addressing decision is made by
the external system. This avoids a call involving I/O during an open
transaction, which can lead to deadlock issues due to a MySQL connector flaw.

This goes beyond the IPAM functions, of course. The idea here being
that all the plugins still need the core data in the Neutron DB even if they
may need to store additonal data or perform additional actions during these
calls.

Out-of-Scope Items
------------------
Several related functions have been discussed in relation to this blueprint.

DHCP options such as nameservers and host routes are intentionally de-coupled
from the IPAM implementation. Pluggable DHCP would require a separate effort,
or must be addressed within the individual drivers.

Similarly, integrations with Designate or other external DNS services during
IPAM activities is out-of-scope.

IPAM Regional Internet Registries (RIRs) life-cycle management and automation
is beyond the scope of this blueprint.


Data Model Impact
-----------------

There will be no data model changes for this implementation, only the addition
of interfaces and non-persistent classes. Rather, related data model updates
are captured in [#]_, though this is not strictly required for this blueprint.

.. [#] https://blueprints.launchpad.net/neutron/+spec/subnet-allocation


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

IPAM subsystem implementation of the default Neutron driver should have
similar performance to the current Neutron IPAM. The performance impact of
external IPAM drivers is beyond the scope of this document.


IPv6 Impact
-----------

Support for IPv6 is a requirement of this specification.


Other Deployer Impact
---------------------

A new configuration option to specify the desired IPAM driver will be available
in the neutron.conf file. If this value is not specified Neutron Server will
fallback to the default Neutron IPAM driver in the default location. This
choice was made to support backward compatibility with older neutron.conf files
that do not have this option specified.


Developer Impact
----------------

* By default the Neutron should work as it does today. Supplied reference IPAM
  driver should encapsulate current functionality.

* As core plugins override several methods from the base plugin class, we will
  evaluate impact of the IPAM changes to those plugins.

Community Impact
----------------

This change was discussed at the Juno and Kilo Design summits. There was
support for Pluggable IPAM, see link to the Etherpad in the Reference
section of document.


Alternatives
------------

None.

Developer Impact
----------------

Implementation
==============

Assignee(s)
-----------

Primary assignee:
 John Belamaric (jbelamaric)

Other contributors:
 Salvatore Orlando (salvatore-orlando)
 Carl Baldwin (carl-baldwin)
 Ryan Tidwell (ryan-tidwell)
 Hosung Hwang (hhwang-2)
 Yue Ko (yko)
 Pavel Bondar (pasha117)

Work Items
----------

  1. Create IPAM abstract interfaces.
  2. Create pluggable IPAM for db_base_plugin_v2.
  3. Move all the IPAM-related functionality from db_base_plugin_v2 to the
     Neutron IPAM plugin.

Dependencies
============

* https://blueprints.launchpad.net/neutron/+spec/reference-ipam-driver

Testing
=======

The existing unit test will be used when appropriate and unit test coverage
will be expanded to cover refactored Neutron IPAM code.

Functional Tests
----------------

Existing Functional Tests will be used when appropriate, refactoring of IPAM
may require additional or refactored functional tests.

Tempest Tests
-------------

The existing Neutron Tempest tests will be utilized to test the default Neutron
IPAM that will be developed.


API Tests
---------

No change to API proposed.


Documentation Impact
====================

User Documentation
------------------

Admin guide will be updated.


Developer Documentation
-----------------------

API guide will be updated.


References
==========

* https://etherpad.openstack.org/p/neutron-ipam
* https://blueprints.launchpad.net/neutron/+spec/subnet-allocation
* https://review.openstack.org/#/c/134339/

