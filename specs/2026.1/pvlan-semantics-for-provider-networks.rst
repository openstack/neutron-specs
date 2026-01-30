..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
PVLAN Semantics for Provider Networks
======================================

RFE: https://bugs.launchpad.net/neutron/+bug/2138746

Private VLAN (PVLAN) [1]_ is a device isolation mechanism through the
application of special Layer 2 forwarding constraints.

This spec describes how to implement the PVLAN mechanism by creating a service
plugin on Neutron, that can be extensible to any ML2 mechanism driver, but that
will be mainly focused on the implementation of the ML2/OVN logic.

Problem Description
===================

Current implementations of Security Groups or other networking isolation tools
do not comply with this security-focused approach to device isolation.  PVLAN
is an established mechanism [1] that has well-defined boundaries and rules for
consistent networking management. There is currently not a straight-forward
way of replicating these boundaries and rules to achieve a consistent
implementation of PVLAN in Neutron with existing plugins and/or extensions. A
mix of security groups and security group rules could make a certain
replication but it would not be flexible or robust enough compared to
implementing its own mechanism.

What PVLAN proposes is a consistent semantic model [2]_ [3]_ to enforce port
isolation rules:

Network attributes
------------------

* **pvlan** `{{enabled | disabled}}`: enable PVLAN semantics for a network.

Port attributes
---------------
* **Roles** for ports (Enforced label if pvlan is enabled):

    + **Promiscuous:** Can reach any other port.
    + **Community:** Can reach only ports within their community + promiscuous.
      One port can only be part of one community.
    + **Isolated:** Can only reach promiscuous, if any.

* **Community label** (if applies) `{{ string }}`: To subgroup community ports

In the following diagram taken from the OVN proof-of-concept [4]_ we can
understand how the different VMs would be configured depending on their role.

.. code-block:: none

    ┌────────────────────────────────────────────────────────────────────────┐
    │                                                                        │
    │                      ┌─────────────────────┐                           │
    │                      │   Promiscuous       │ ← Can communicate with    │
    │                      │   192.168.1.30      │   ALL ports               │
    │                      └──────────┬──────────┘                           │
    │                                 │                                      │
    │           ┌─────────────────────┼──────────────────────┐               │
    │           │                     │                      │               │
    │           ▼                     ▼                      ▼               │
    │    ┌──────────────────┐ ┌──────────────────┐    ┌──────────────────┐   │
    │    │  Isolated        │ │  Community-1     │    │  Community-2     │   │
    │    │                  │ │                  │    │                  │   │
    │    │  ┌────┐  ┌────┐  │ │  ┌────┐  ┌────┐  │    │  ┌────┐  ┌────┐  │   │
    │    │  │iso1│  │iso2│  │ │  │1-c1│◄►│2-c1│  │    │  │1-c2│◄►│2-c2│  │   │
    │    │  │.1  │  │.2  │  │ │  │.10 │  │.11 │  │    │  │.20 │  │.21 │  │   │
    │    │  └────┘  └────┘  │ │  └────┘  └────┘  │    │  └────┘  └────┘  │   │
    │    │                  │ │                  │    │                  │   │
    │    └──────────────────┘ └──────────────────┘    └──────────────────┘   │
    │   (Can only talk          (Members can talk     (Members can talk      │
    │    to Promiscuous)         within community      within community      │
    │                            + Promiscuous)        + Promiscuous)        │
    │                                                                        │
    │                                                                        │
    └────────────────────────────────────────────────────────────────────────┘

Proposed Change
===============

A new service plugin of Neutron that will implement new API extension and will
be extensible by the various backend drivers.

.. note:: The scope of this spec is to complete the driver implementation of
   ML2/OVN. Any interested contributor is welcomed to implement the ML2/OVS
   backend.

REST API Impact
---------------

As described before, there are new attributes present on both networks and
ports.

New table **networkpvlan** extending networks:

* ``network_id`` - `UUID type`.
* ``pvlan`` - `boolean type`. True if PVLAN is enabled. Default: False.

Example:

+--------------------------------------+-------+
| network_id                           | pvlan |
+======================================+=======+
| 2c1b70d6-1ffb-43e8-8069-53f5597c446e |     0 |
+--------------------------------------+-------+

.. code::

    POST /v2.0/networks
    Accept: application/json

    {
    "network": {
        "name": "pvlan-net-1",
        "admin_state_up": true,
        "pvlan": true}
    }


Response:

.. code::

   {
    "network": {
        "id": "a9254bdb-2613-4a13-ac4c-adc581fba50d",
        "name": "pvlan-net-1",
        "status": "ACTIVE",
        "subnets": [],
        "admin_state_up": true,
        "shared": false,
        "project_id": "fb0e16da8ce7435c8a444647738e2166",
        "pvlan": true,
        "tenant_id": "fb0e16da8ce7"
        }
    }


New table **portpvlan** extending ports:

* ``port_id`` - `UUID type`.

* ``pvlan_type`` - `string type`. Allowed values: "isolated", "community",
  "promiscuous". Default: Promiscuous.

* ``pvlan_community`` - `string type`. Allowed values: Any string that complies
  with Port Group name restriction notes in OVN [5]_ — Names are ASCII and must
  match ``[a-zA-Z_.][a-zA-Z_.0-9]*``. Default: None.

Example:

+--------------------------------------+------------+-----------------+
| port_id                              | pvlan_type | pvlan_community |
+======================================+============+=================+
| 7abca15f-a19a-4f4d-a27c-b695ca1ec44b | community  | community_1     |
+--------------------------------------+------------+-----------------+
| e66abfad-a681-4a90-a6f7-8efe5e4bd2ae | isolated   | NONE            |
+--------------------------------------+------------+-----------------+


.. code::

    POST /v2.0/ports
    Accept: application/json

    {
    "port": {
        "name": "pvlan-port-1",
        "network_id": "03848825-c4fc-44c1-b510-fc6bf64cc8ef",
        "pvlan_type": "community",
        "pvlan_community": "community_1"}
    }

Response:

.. code::

    {
    "port": {
        "id": "e66abfad-a681-4a90-a6f7-8efe5e4bd2ae",
        "name": "port_2",
        "network_id": "03848825-c4fc-44c1-b510-fc6bf64cc8ef",
        "mac_address": "fa:16:3e:a8:70:16",
        "status": "DOWN",
        "device_id": "f51da0f6-f1e2-48cb-b46d-d37a262f383e",
        "pvlan_type": "community",
        "project_id": "fb0e16da8ce7435c8a444647738e2166",
        "pvlan_community": "community_1"}
    }


Sub Resource Extension
----------------------

It will be created as an extension that extends the parameters for network and
for port:

.. code-block:: python

    RESOURCE_ATTRIBUTE_MAP = {
        network.COLLECTION_NAME: {
            pvlan_const.PVLAN_MODE: {
                'allow_post': True,
                'allow_put': True,
                'is_visible': True,
                'default': False,
                'enforce_policy': True,
                'validate': {'type:boolean': None},
                'convert_to': converters.convert_to_boolean}
        },
        port.COLLECTION_NAME: {
            pvlan_const.PVLAN_TYPE: {
                'allow_post': True,
                'allow_put': True,
                'is_filter': True,
                'default': pvlan_const.PROMISCUOUS_TYPE,
                'is_visible': True,
                'validate': {'type:values': pvlan_const.PVLAN_TYPES}
            },
            pvlan_const.PVLAN_COMMUNITY: {
                'allow_post': True,
                'allow_put': True,
                'is_visible': True,
                'default': None,
                'enforce_policy': True,
                'validate': {
                    'type:regex_or_none': pvlan_const.COMMUNITY_NAME_REGEX}
            }
        }
    }

Service Plugin
--------------

The new service plugin ``pvlan`` will implement the API extension and will
extend Neutron resources with the attributes needed for port and network. It
will also define the new DB objects and will call the backend driver for the
attribute operations (create / update / delete) implementation.


ML2/OVN Driver Implementation details
-------------------------------------

The backend driver needs to implement the operations by consuming Neutron
notifications about creation, deletion and update of the Networks and Ports.

The operations to be implemented are:

* **Network create (enable pvlan)** -

        1. Create ``pvlan_drop_all``, ``pvlan_isolated`` and
           ``pvlan_promiscuous`` Port Groups for the network (Logical Switch).
        2. Create ACLs for those Port Groups.

Port Group example:

.. code:: sh

    _uuid               : 6dfac905-892c-44f6-880c-7168399e7240
    acls                : [f21e358d-ce0b-4741-80bf-a13abeefc221, 41cfb444-c744-4d4f-be2f-706f17bab37e]
    external_ids        : {"neutron:network"="cb33079f-9309-4312-bd0f-9d57d20c8c0a"}
    name                : pg_pvlan_community_1
    ports               : [e9c40083-d420-47c9-86ec-d4de363d8c1b]


ACL example:

.. code:: sh

    _uuid               : f21e358d-ce0b-4741-80bf-a13abeefc221
    action              : allow-stateless
    direction           : to-lport
    external_ids        : []
    label               : 0
    log                 : false
    match               : "outport == @pg_pvlan_community_1 && inport == @pg_pvlan_community_1 && (ip || arp)"
    meter               : []
    name                : []
    options             : {}
    priority            : 1010
    severity            : []
    tier                : 0


* **Network delete (disable pvlan)** - delete
  ``pvlan_drop_all``, ``pvlan_isolated`` and ``pvlan_promiscuous`` Port Groups
  for the network (Logical Switch), delete ACLs created for those Port Groups.

* **Port create** -

    * If **pvlan_type=isolated** or **pvlan_type=promiscuous** - add port to
      the specific port group.

    * If **pvlan_type=community** - ensure that Port Group and ACLs for that
      Port Group are created and add port to that Port Group.

* **Port update** - Depending on the updated value, remove port from one Port
  Group and add to another.

* **Port delete** - Delete port from the corresponding Port Group.  If
  **pvlan_type=community** and it is the last port in the Port Group
  corresponding to that community name, **remove that Port Group as well**.


In ML2/OVN Port Groups can serve as a tool for defining Communities within a
PVLAN domain. In this implementation there would also be a specific Port group
for ports from isolated VMs, and another one for ports from promiscuous VMs.
Each Port group will be then associated with ACLs that will define their
behaviour.

Since ACLs have a priority parameter, we can tune them so that they will
interact as intended e.g. right now the 'neutron_pg_drop' related ACLs have
lower priority than the security group ACLs. With PVLAN the pg_drop_group would
still exist (or we would create a similar one just for PVLAN logic, depending
on what is best). This Port group would have the lowest priority. Security
group rules could coexist with PVLAN by placing on PVLAN a higher priority on
its ACLS, making the PVLAN ACLs be the ones to be used if they exist.

We don't need to track state of the connection so ACLs actions will be
stateless, which is better in terms of the performance in some cases.

.. note:: Ports with disabled port security in the network will be forbidden if
   pvlan is enabled, as there is no a better solution that we can think of for
   its coexistence.

.. warning:: Since the implementation in ML2/OVN uses ACLs, there is
   no compatibility with external PVLAN networks.

Remaining questions on Implementation
-------------------------------------

* Should we disable arp_responder for LSP to be able to block that traffic too?
  Or maybe it is possible to configure OVN to respond to ARP traffic
  “selectively”?

* Would pg_drop_group be different to the general drop port group of PVLAN or
  can we use the same one?


Upgrading
---------

This feature does not affect how current components work, so no extra effort
should be done regarding upgrades.

Implementation
==============

Assignee(s)
-----------

* Slawek Kaplonski
* Elvira García

Work Items
----------
* API Definition
* Service implementation
* ML2/OVN logic implementation

Testing
=======

* Unit/functional tests.
* Scenario test in neutron tempest.

Documentation
=============

* User Documentation: How to set up PVLAN in Neutron with ML2/OVN

References
==========

.. [1] https://datatracker.ietf.org/doc/html/rfc5517
.. [2] https://www.cisco.com/c/en/us/support/docs/lan-switching/private-vlans-pvlans-promiscuous-isolated-community/40781-194.html
.. [3] https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus5000/sw/layer2/503_n2_1/503_n2_1nw/Cisco_n5k_layer2_config_gd_rel_503_N2_1_chapter5.pdf
.. [4] https://gist.githubusercontent.com/gprocunier/91ad630db6e573b966ed4008dfa79bb8/raw/0e4ce3243eba7679c6a6093414a9286808bdf40d/pvlan_poc.sh
.. [5] https://www.ovn.org/support/dist-docs/ovn-nb.5.html
