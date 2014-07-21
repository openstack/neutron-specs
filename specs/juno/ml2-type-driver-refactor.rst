================================================
ML2 Type drivers refactor to allow extensiblity
================================================

Blueprint URL:

https://blueprints.launchpad.net/neutron/+spec/ml2-type-driver-refactor

This blueprint aims to refactor the ML2 type driver architecture
such that type drivers are more self sufficient and allows developers
to author custom type drivers.

Flows

Flows represented are only a partial flow of events relevant to this patch

Current Flow:

.. seqdiag::

  seqdiag {
    API -> Neutron [label = "POST create_network"];
    Neutron -> ML2_Plugin [label = "create_network"];
    ML2_Plugin -> Type_Manager [label = "create_network"];
    Type_Manager -> VLAN_Driver [label = "allocate_tenant_segment"];
    Type_Manager -> VxLAN_Driver [label = "allocate_tenant_segment"];
    Type_Manager <-- VLAN_Driver [label = "segment"];
    Type_Manager <-- VxLAN_Driver [label = "segment"];
    ML2_Plugin <-- Type_Manager [label = "segments"];
    Neutron <-- ML2_Plugin [label = "Network dictionary"];
    API <-- Neutron [label = "Network dictionary"];
  }

Problem description
===================
Currently the ML2 segmentation is managed in multiple places within
the architecture:

* The plugin invokes the type manager with tenant/provider segment calls.
* The manager in turn invokes the type driver to reserve an available/specified
  segment for the network.
* The segment_id if reserved is returned all the way to the plugin which then
  stores it inside a DB.
* The type driver itself also has tables of its own where it stores
  segment details.

This model works well for generic segmentation types like vlan, gre etc.
But this also makes any network types deviating from the norm like dynamic
network segments near impossible to implement.

Use Case

A sample use case for this feature is an overlay network with VTEP’s in the
hardware layer instead of the soft switch edge. To overcome the 4k VLAN limit
a single network can have different segmentation ID’s depending on location
in the network. The assumption that a network has fixed segmentation id’s
across the deployment doesn’t work with this use case with external
controllers that are capable of managing this kind of segmentation.


Proposed change
===============

* Move segmentation logic to the type managers and type drivers, send network
  dictionary to the type drivers to store network information. This involves
  moving the segmentation management bits from the ML2 plugin to the type
  manager and the type drivers will now require the network dictionary to be
  passed when allocating or reserving a segment.
* Add a new optional method called allocate_dynamic_segment to be invoked from
  the mechanism drivers and consumed by the type drivers if required. This call
  will typically be invoked by the mechanism driver in it's bind_port method,
  it can also be invoked at any time if so desired by the mechanism driver(s).
* Store provider information in the DB tables, this is just a boolean that
  distinguishes between a provider network and a non-provider network. This
  information is required by many vendor plugins that wish to differentiate
  between a provider and non-provider network.
* Agent interactions for the dynamic segments will be handled by Bob Kukura's
  patch: https://blueprints.launchpad.net/neutron/+spec/ml2-dynamic-segment

Proposed Flows:

.. seqdiag::

  seqdiag {
    API -> Neutron [label = "POST create_network"];
    Neutron -> ML2_Plugin [label = "create_network"];
    ML2_Plugin -> Type_Manager [label = "create_network"];
    Type_Manager -> Type_Driver [label = "allocate_segment(static)"];
    Type_Manager <-- Type_Driver [label = "static segment"];
    ML2_Plugin <-- Type_Manager [label = "segments(static)"];
    Neutron <-- ML2_Plugin [label = "Network dictionary"];
    API <-- Neutron [label = "Network dictionary"];
  }

.. seqdiag::

  seqdiag {
    ML2_Plugin -> Mech_Manager [label = "bind_port"];
    Mech_Manager -> Mech_Driver [label = "bind_port"];
    Mech_Driver -> Type_Manager [label = "allocate_segment(dynamic)"];
    Type_Manager -> Type_Driver [label = "allocate_segment(dynamic)"];
    Type_Manager <-- Type_Driver [label = "segment(dynamic)"];
    Mech_Driver <-- Type_Manager [label = "segments(dynamic)"];
    Mech_Manager <-- Mech_Driver [label = "Bindings(dynamic)"];
    ML2_Plugin <-- Mech_Manager [label = "Bindings(dynamic)"];
  }

Alternatives
------------
The alternative is to override the get_device_details rpc method and send
the RPC call from the agent to the type or mechanism driver and serve a vlan
out of a pool. This approach will require extensive changes to the rpc
interactions to send the calls directly to the type or mechanism drivers
for them to override. Also, having mechanism drivers do segment management
will break the ml2 model of clean separation between type and mechanism
drivers.

Data model impact
-----------------
The data model for all the type drivers will change to accomodate network id's.
This translates to one extra column in the database for each type driver to
store the network uuid. Existing models for ml2_network_segments will be
modified to only store the type driver type and not the complete segment.

The patch will be accompanied by an Alembic migration to upgrade/downgrade all
type driver and ML2 DB models.

NetworkSegment:

Removed Fields:
network_type = sa.Column(sa.String(32), nullable=False)
physical_network = sa.Column(sa.String(64))
segmentation_id = sa.Column(sa.Integer)

Added Fields:
segment_type = sa.Column(sa.String(32), nullable=False)

Add DB based type drivers (vlan/vxlan/gre):
Added Fields:
network_id = sa.Column(sa.String(255), nullable=True)
provider_network = sa.Column(sa.Boolean, default=False)

As part of the Alembic migration script, data will also be migrated
out of the NetworkSegment tables to the respective type driver tables
and vice versa for upgrades/downgrades.

As of today there is no method of knowing if the existing segment is a
provider segment or allocated tenant segment in ML2 so all migrated networks
will appear as tenant segments.

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
Minor performance impact as extra information will be sent to the type drivers
to store in their database.

Other deployer impact
---------------------
None.

Developer impact
----------------
No API impact. Developers developing new type drivers will have to
manage all segmentation within the type driver itself.


Implementation
==============

Assignee(s)
-----------
Arvind Somya <asomya>

Work Items
----------
* Modify ML2 plugin.py to remove all segmentation management.
* Modify ML2 network_segments model to only store type drivers for
  each network.
* Move segment processing to type manager.
* Modify type manager to handle getting segments.
* Modify type manager to send network dictionary to the type drivers.
* Refit type driver models to store network id's.
* Implement dynamic allocation method in base plugin.
* Implement dynamic allocation method in all existing type drivers.
* Modify existing unit tests for the new architecture.
* Add unit tests for dynamic allocation.


Dependencies
============
None.


Testing
=======
Complete unit test coverage for the ML2 plugin and all refitted type drivers.


Documentation Impact
====================
None.


References
==========
None.
