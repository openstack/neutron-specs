..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================================
ML2/OVN - Coexistence Support for OVN externally managed resources
==================================================================

https://bugs.launchpad.net/neutron/+bug/2027742

Currently two individual Neutron deployments using ML2/OVN are separate and may
only communicate using normal provider networks. However OVN supports the
feature ``OVN interconnect`` (OVN-IC), that allows multiple separate OVN
deployments to be connected together. The interconnection is implemented using
a normal overlay (just like the one between compute nodes) and can therefore be
created easily in a large scale (something that is not necessarily the case for
provider networks).

Nowadays the Neutron OVN db sync command is actively interfering with
``OVN interconnect`` by removing non-Neutron resources from the Northbound
database. The goal of this spec is to remove this interference and allow
operators to use OVN interconnect and other features externally managed by OVN.
It is out of the scope of this spec to integrate OVN interconnect directly to
Neutron (however this might be part of a future spec).

Background: OVN Interconnect use case
=====================================

The OVN Interconnect allows multiple clusters to be interconnected at Layer 3
level. This can be useful for deployments with multiple Availability
Zones (AZs) that needs to allow connectivity between workloads in separate
zones.

In the case of layer 3 interconnection, the logical routers on each cluster /
availability zone can be connected via transit overlay networks. The transit
network is an abstract representation of the interconnect layer, and the
element responsible for this intercluster visibility is the Transit Switch (TS)
. These interconnection switches are created in a global database and
replicated to each AZ via OVN-IC daemon. So, basically, a logical router in one
AZ is connected to a Logical router in another AZ via Transit switch. More
details can be found in the ovn-architecture manpage [1]_.

The reference design using two separated OVN clusters (north and south) is
described below::

 +---------------------------------------------------------------------+
 |OVN north cluster                                                    |
 |                        +------------------------------------------+ |
 |                        |OVN central node                          | |
 | +---------------+      |  +-------------+        +-------------+  | |
 | |network node   |      |  |ovsdb-server |        |ovsdb-server |  | |
 | |ovn-controller +---------+southbound   |        |northbound   |  | |
 | |               |      |  |             |        |             |  | |
 | +---------------+      |  |             |        |             |  | |
 |                        |  |             |        |             |  | |
 | +---------------+      |  |             |        |             |  | |
 | |compute node   |      |  |             |        |             |  | |
 | |ovn-controller +---------+             |        |             |  | |
 | |               |      |  |             |        |             |  | |
 | +---------------+      |  +------+------+        +-------+-----+  | |
 |                        |         |                       |        | |
 |                        |         +---+---------------+---+        | |
 |                        |             |               |            | |
 |                        |  +----------+-----+   +-----+---------+  | |
 |                        |  |ovn-ic          |   |ovn-northd     |  | |
 |                        |  |                |   |               |  | |
 |                        |  |                |   |               |  | |
 |                        |  +-------+--------+   +---------------+  | |
 |                        |          |                               | |
 |                        +----------|-------------------------------+ |
 |                                   |                                 |
 +-----------------------------------|---------------------------------+
                                     |
             +-----------------------+---------------------+
             |Global DB - Transit switches                 |
             |ovsdb-server: northbound IC, southbound IC   |
             |                                             |
             +-----------------------+---------------------+
                                     |
 +-----------------------------------|---------------------------------+
 |OVN south cluster                  |                                 |
 |                        +----------|-------------------------------+ |
 |                        |          |                               | |
 |                        |  +-------+--------+   +---------------+  | |
 |                        |  |ovn-ic          |   |ovn-northd     |  | |
 |                        |  |                |   |               |  | |
 |                        |  |                |   |               |  | |
 |                        |  +----------+-----|   +-----+---------+  | |
 |                        |             |               |            | |
 |                        |         +---+---------------+---+        | |
 |                        |         |                       |        | |
 | +---------------+      |  +------+------+        +-------+-----+  | |
 | |network nodeer |      |  |ovsdb-server |        |ovsdb-server |  | |
 | |ovn-controller +---------+southbound   |        |northbound   |  | |
 | |               |      |  |             |        |             |  | |
 | +---------------+      |  |             |        |             |  | |
 |                        |  |             |        |             |  | |
 | +---------------+      |  |             |        |             |  | |
 | |compute node   |      |  |             |        |             |  | |
 | |ovn-controller +---------+             |        |             |  | |
 | |               |      |  |             |        |             |  | |
 | +---------------+      |  +-------------+        +-------------+  | |
 |                        |OVN central node                          | |
 |                        +------------------------------------------+ |
 +---------------------------------------------------------------------+

The following example will outline how OVN Interconnect works within the scope
of OVN. It therefore follows the naming of OVN and not that of Neutron (should
the two disagree).

Example setup::

 +-------------------------------------------------------------+
 |                     Logical_Switch                          |
 |                     ts1                                     |
 +----------+----------------------------------+---------------+
            |                                  |
    +-------+-----------+              +-------+-----------+
    |Logical_Switch_Port|              |Logical_Switch_Port|
    |lsp_ts1_lr1        |              |lsp_ts1_lr2        |
    +-------+-----------+              +-------+-----------+
            |                                  |
    +-------+------------+             +-------+------------+
    |Logical_Router_Port |             |Logical_Router_Port |
    |lrp_lr1_ts1         |             |lrp_lr2_ts1         |
    |172.24.0.10/24      |             |172.24.0.20/24      |
    +-------+------------+             +-------+------------+
            |                                  |
    +-------+------------+             +-------+------------+
    |Logical_Router      |             |Logical_Router      |
    |lr1                 |             |lr2                 |
    +-------+------------+             +-------+------------+
            |                                  |
    +-------+------------+             +-------+------------+
    |Logical_Router_Port |             |Logical_Router_Port |
    |lrp_lr1_ls1         |             |lrp_lr2_ls2         |
    |192.168.0.1/24      |             |192.168.1.1/24      |
    +-------+------------+             +-------+------------+
            |                                  |
    +-------+------------+             +-------+------------+
    |Logical_Switch_Port |             |Logical_Switch_Port |
    |lsp_ls1_lr1         |             |lsp_ls2_lr2         |
    +-------+------------+             +-------+------------+
            |                                  |
    +-------+------------+             +-------+------------+
    |Logical_Switch      |             |Logical_Switch      |
    |ls1                 |             |ls2                 |
    +-------+------------+             +-------+------------+
            |                                  |
    +-------+------------+             +-------+------------+
    |Logical_Switch_Port |             |Logical_Switch_Port |
    |lsp_ls1_vm1         |             |lsp_ls2_vm2         |
    +--------------------+             +--------------------+


The example above is a logical representation of the elements involved in the
interconnection process. On each side we have an OVN cluster/AZ with its local
managed resources: LSP for VMs, LS, LSP connecting the VM to the router and
the LR. What does OVN interconnect add to a standard topology to make this
work? A connection between the Tenant logical router and a Transit Switch.

The global database of the OVN IC dynamically replicates the TS between all
members of the interconnect domain (clusters/AZs), and what needs to be done
is basically to add this dynamically created TS in the OVN Northbound database
with the Tenant logical router.


Ownership of resources
======================

With the usage of OVN Interconnect Neutron is no longer the only owner of
resources in each OVN deployment.

We therefore need to first define which component owns which kind of resources.
The resources will be listed below for the left OVN deployment from the
example above.

Resources owned by Neutron
--------------------------
* lr1
* lrp_lr1_ls1
* lsp_ls1_lr1
* ls1
* lsp_ls1_vm1

All of these represent the:
* Tenant Network (ls1)
* Port of VMs or other things (lsp_ls1_vm1)
* Router of the Tenant (lr1, lrp_lr1_ls1, lsp_ls1_lr1)

Resources owned by OVN-IC
-------------------------
* ts1
* lsp_ts1_lr2 (created in the left OVN deployment)
* potentially Logical_Router_Static_Routes attached to lr1
  (if route learning is enabled)

The resources owned by OVN-IC are dynamically created and removed by the
OVN-IC daemon when the process synchronizes the Northbound database between
interconnect domain elements.

resources owned by the operator
-------------------------------
* lsp_ts1_lr1
* lrp_lr1_ts1

The resources to connect the Transit Switch to the router of the user need to
be created by the operator (manually or with some kind of automation).
Managing these resources in Neutron is out of scope of this spec.

Problem Description
===================

The above setup can already be created by an operator.
However the `neutron-ovn-db-sync-util` tool will remove the resources owned by
OVN-IC and the operator, as Neutron does not know about them.

Example of resources created by OVN-IC:

* Logical_Switch
  `other_config:interconn-ts` with any value
* Logical_Router_Static_Route
  `external_ids:ic-learned-route` with any value
* Logical_Switch_Port
  `type` is set to `remote`

These fields are automatically set by OVN-IC, so they do not have a Neutron key
in the `external_ids` or `other_config` registers.

Example of resources owned by the operator:

* Logical_Switch_Port
  `external_ids` or `other_config` with any value
* Logical_Router_Port
  `external_ids` or `other_config` with any value

For resources created by operators such predefined options for `external_ids`
or `other_config` do not exist.


Proposed Change
===============

To solve the problem described above, the proposal is to introduce a new filter
rule to check for the Neutron key during the `neutron-ovn-db-sync-util` method
and not remove resources externally managed by OVN.

This implementation is being named as to `coexistence support for OVN
externally managed resources` because it is out of scope any type of OVN
externally managed resources integration as part of Neutron. The proposal of
this implementation is the creation of filters in the checking of the resources
created by Neutron, and it is the basis for any future implementation that
intends to integrate the OVN interconnect or other OVN features to Neutron.

To implement the coexistence support the `neutron-ovn-db-sync-util` tool only
needs to check the resources managed by Neutron. The main idea of this proposal
is described below.

For resources created by Neutron the proposed solution implements the support
by checking the specific Neutron signature on these resources. Neutron creates
resources in the OVN NB database with the `neutron:` key in the `external_ids`
register:

* Logical_Switch
  `external_ids:"neutron:"` with any value

.. code::

  _uuid               : 5bf82c8e-fa49-4b46-bc4f-737311359f44
  acls                : []
  copp                : []
  dns_records         : []
  external_ids        : {"neutron:availability_zone_hints"="",
                         "neutron:mtu"="1442",
                         "neutron:network_name"=self_network_az1_tenant1,
                         "neutron:revision_number"="2"}
  forwarding_groups   : []
  load_balancer       : []
  load_balancer_group : []
  name                : neutron-2979fcc1-9540-4012-88a2-5f83738b5b6f
  other_config        : {mcast_flood_unregistered="false", mcast_snoop="false",
                         vlan-passthru="false"}
  ports               : [8e128001-5d9a-41eb-a84b-052dc28a74bc,
                         98555f1c-1a12-4703-8dcb-67f44900f6b8,
                         abeca18e-5706-4a2d-871d-5514ba20f554,
                         ba0e5d5e-5571-4f8c-b15c-dfa5115ba61c]
  qos_rules           : []

* Logical_Switch_Port, Logical_Router, Logical_Router_Port, etc.
  `external_ids:"neutron:"` with any value

These `neutron:...` keys in the external_ids are automatically set by Neutron,
so we can rely on them being there. We need to ensure that all methods called
by `neutron-ovn-db-sync-util` check the Neutron signature in the external_ids,
and filter/ignore all the other externally managed resources when synchronizing
between Neutron and the OVN NB database.

DB Impact
---------

None

Rest API Changes
----------------

None

OVN driver changes
------------------

Update `neutron-ovn-db-sync-util` as described above.

Out of Scope
============

* Integrating ovn-interconnect or other OVN externally managed feature into
  Neutron in any way;

Implementation
==============

Assignee(s)
-----------

* Primary assignees:
  Felix Huettner <felix.huettner@mail.schwarz>
  Roberto Bartzen Acosta <rbartzen@gmail.com>

Work Items
----------

* Add exclusion to `neutron-ovn-db-sync-util`

* Implement relevant unit and functional tests using the existing facilities
  in Neutron.

* Write documentation.

Documentation Impact
====================

Administrator Documentation
---------------------------

Administrator documentation will need be included to describe to operators how
to use the OVN interconnect when building their interconnected OpenStack
clusters.

Testing
=======

* Unit/functional tests [2]_.

References
==========

.. [1] https://www.ovn.org/support/dist-docs/ovn-architecture.7.html
.. [2] https://docs.openstack.org/neutron/latest/contributor/testing/testing.html
