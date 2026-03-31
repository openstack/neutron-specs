..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
L3 Agent Scheduler API Support for ML2/OVN
==========================================

https://bugs.launchpad.net/neutron/+bug/2103521

The L3 Agent Scheduler API (``l3_agent_scheduler`` extension) currently only
works with the classic L3 agent (``neutron-l3-agent``) backed by the
``RouterL3AgentBinding`` database table. ML2/OVN deployments use an entirely
different scheduling model based on OVN ``HA_Chassis_Group`` and
``HA_Chassis`` rows in the Northbound database. This leaves operators of
ML2/OVN clouds without an API to query which gateway chassis host a given
router, or which routers are scheduled on a given chassis.

This spec proposes to implement the full L3 Agent Scheduler API for
the ML2/OVN mechanism driver so that operators can query, create, and
remove the associations between Neutron routers and their gateway
chassis, along with the ``HA_Chassis`` priority that governs failover
order. A new API extension will expose the priority information, which
is not present in the current API, and will also allow specifying a
priority when scheduling a router onto a chassis.


Problem Description
===================

Use cases
---------

* **Operator visibility (router → chassis):** an operator wants to know
  which gateway chassis are assigned to a Neutron router, what their
  ``HA_Chassis`` priorities are, and which chassis is currently the
  primary (highest priority) gateway. Today this information is only
  available through direct ``ovn-nbctl`` commands or OVSDB queries, not
  through the Neutron API.

* **Operator visibility (chassis → routers):** an operator plans to drain
  or maintain a gateway chassis and needs to list all routers that have
  gateway ports scheduled on that chassis.

* **Tooling and dashboards:** monitoring and capacity-planning tools need
  a standard API to collect gateway chassis assignments, including their
  priorities, across the fleet. The ``HA_Chassis`` priority determines
  failover order and is essential for understanding the HA posture of each
  router.

* **Parity with ML2/OVS:** the L3 Agent Scheduler API is well established
  for ML2/OVS deployments. ML2/OVN operators and tools that already consume
  ``GET /v2.0/routers/{router_id}/l3-agents`` or
  ``GET /v2.0/agents/{agent_id}/l3-routers`` should be able to use the same
  endpoints to obtain equivalent information without being aware of the
  underlying backend.

* **Manual scheduling control:** an operator wants to pin a router's
  primary gateway to a specific chassis (e.g. for maintenance windows,
  capacity planning, or compliance) or add/remove backup chassis from
  the ``HA_Chassis_Group``. Today this requires direct OVSDB manipulation.

* **Priority-aware scheduling:** when manually assigning a chassis to a
  router, the operator needs to control the ``HA_Chassis`` priority to
  decide whether the chassis becomes the new primary (highest priority)
  or a specific backup slot.


Proposed Change
===============

Overview
--------

The change bridges the conceptual gap between the classic L3 Agent Scheduler
model (routers ↔ L3 agents via ``RouterL3AgentBinding``) and the OVN model
(gateway ``Logical_Router_Port`` ↔ ``HA_Chassis_Group`` ↔ ``HA_Chassis``).

The implementation covers all four operations of the L3 Agent Scheduler
API: listing agents hosting a router, listing routers on an agent,
scheduling a router onto an agent (with optional priority), and removing
a router from an agent.

In ML2/OVN, each gateway chassis is represented by an ``OVN Controller
Gateway agent`` row in the Neutron ``agents`` table. The router scheduling
information lives in OVN Northbound DB as ``HA_Chassis`` rows (each
carrying a ``chassis_name`` and a ``priority``). Since [1]_, gateway
``Logical_Router_Port`` scheduling was migrated from the legacy
``Gateway_Chassis`` table to the ``HA_Chassis_Group`` / ``HA_Chassis``
tables, which is the current and preferred mechanism.

.. note::

   The ``HA_Chassis.priority`` field in the OVN Northbound schema is
   defined as an integer in the range 0–32767 [2]_. Although the schema
   allows this wide range, Neutron currently limits the number of chassis
   in an ``HA_Chassis_Group`` to ``MAX_GW_CHASSIS`` (5) and assigns
   priorities within that smaller range.


Mapping of OVN concepts to the L3 Agent Scheduler API
-----------------------------------------------------

.. list-table::
   :header-rows: 1

   * - Classic L3 Agent model
     - OVN equivalent
   * - L3 Agent (``neutron-l3-agent``)
     - OVN Controller Gateway agent (gateway-capable ``Chassis``)
   * - ``RouterL3AgentBinding``
     - ``HA_Chassis`` within an ``HA_Chassis_Group`` assigned to a gateway
       ``Logical_Router_Port``
   * - Router ↔ Agent association
     - Router ↔ Chassis association via ``Logical_Router_Port`` →
       ``HA_Chassis_Group`` → ``HA_Chassis.chassis_name``
   * - *(no equivalent)*
     - ``HA_Chassis.priority`` (failover order within the group)


GET /v2.0/routers/{router_id}/l3-agents — List agents hosting a router
----------------------------------------------------------------------

For a given router, the implementation will:

#. Look up all gateway ``Logical_Router_Port`` rows for the router in
   the OVN Northbound DB (identified by
   ``external_ids:neutron:router_name``).
#. For each gateway LRP, resolve the ``HA_Chassis_Group`` and iterate
   over the ``HA_Chassis`` rows.
#. Map each ``HA_Chassis.chassis_name`` to the corresponding Neutron
   ``OVN Controller agent`` row in the ``agents`` table.
#. Return the list of agent dicts in the standard ``{"agents": [...]}``
   response format.

When the new ``l3-agent-scheduler-ha-priority`` API extension is loaded
(see below), each agent dict will include an additional
``ha_chassis_priority`` field.


GET /v2.0/agents/{agent_id}/l3-routers — List routers on an agent
-----------------------------------------------------------------

For a given OVN Controller agent (which maps to a gateway ``Chassis``):

#. Resolve the agent to its ``chassis_name``.
#. Use the existing ``get_all_chassis_gateway_bindings()`` method (or
   similar NB IDL query) to find all gateway ``Logical_Router_Port``
   rows that have an ``HA_Chassis`` entry for this chassis.
#. Map each LRP back to the Neutron router UUID (via
   ``external_ids:neutron:router_name``).
#. Return the list of router dicts in the standard
   ``{"routers": [...]}`` response format.


POST /v2.0/agents/{agent_id}/l3-routers — Schedule a router to a chassis
------------------------------------------------------------------------

This operation adds a chassis to the ``HA_Chassis_Group`` of the
router's gateway ``Logical_Router_Port``. The implementation will:

#. Validate that the agent identified by ``agent_id`` is a
   gateway-capable ``OVN Controller Gateway agent`` (i.e. the
   corresponding ``Chassis`` has ``enable-chassis-as-gw`` set and
   appropriate bridge mappings for the router's external network).
#. Validate that the router has a gateway ``Logical_Router_Port``.
#. Validate that the chassis is not already in the router's
   ``HA_Chassis_Group``.
#. Validate that the ``HA_Chassis_Group`` has not reached
   ``MAX_GW_CHASSIS`` (currently 5).
#. Create a new ``HA_Chassis`` row in the ``HA_Chassis_Group`` with
   the appropriate priority.

**Priority handling in the request body:**

When the ``l3-agent-scheduler-ha-priority`` API extension is loaded,
the ``POST`` request body accepts an optional ``ha_chassis_priority``
field alongside the existing ``router_id``:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``router_id``
     - string (UUID)
     - yes
     - The router to schedule onto this agent.
   * - ``ha_chassis_priority``
     - integer
     - no
     - The desired priority for this chassis in the
       ``HA_Chassis_Group``.

The priority behaviour depends on whether ``ha_chassis_priority`` is
supplied:

* **Priority not provided:** the new chassis is assigned the lowest
  priority among all chassis currently in the ``HA_Chassis_Group``. If
  the group is empty, the chassis receives priority 1. This makes the
  new chassis a backup with the lowest failover preference.

  If the ``HA_Chassis_Group`` has a chassis with the lowest priority assigned,
  the API call will fail. The API does not reschedule existing priorities.

* **Priority provided:** the new chassis is inserted at the requested
  priority. If any existing ``HA_Chassis`` register matches the provided
  priority, the API call will fail. This is the same consideration as in the
  previous section: the API does not reschedule the existing priorities. The
  user needs to rebalance manually the chassis and priorities assigned to a
  router.

The request without the extension loaded uses the standard body
``{"router_id": "<uuid>"}`` and the chassis is added at the lowest
priority.


PUT /v2.0/agents/{agent_id}/l3-routers/{router_id} — Update a chassis priority
-------------------------------------------------------------------------------

This operation updates the ``HA_Chassis.priority`` of an existing
chassis assignment for a router. It requires the
``l3-agent-scheduler-ha-priority`` API extension to be loaded; if the
extension is not loaded, the endpoint returns ``404 Not Found``.

This API call is currently not provided by the L3 Agent Scheduler API [3]_
because it wasn't needed to update the ``ha_chassis_priority``. This API
extension will be only supported by the OVN L3 plugin ``OVNL3RouterPlugin``;
it won't be supported by the L3 agent plugin ``L3RouterPlugin``.

The implementation will:

#. Validate that the agent identified by ``agent_id`` is an
   ``OVN Controller Gateway agent``.
#. Validate that the router identified by ``router_id`` has a gateway
   ``Logical_Router_Port`` with an ``HA_Chassis_Group``.
#. Validate that the chassis is already present in the router's
   ``HA_Chassis_Group``. If it is not, the call returns
   ``409 Conflict``.
#. Validate that no other ``HA_Chassis`` in the group already holds the
   requested priority. If a conflict exists, the call returns
   ``409 Conflict`` — the API does not reschedule existing priorities.
#. Update the ``HA_Chassis.priority`` of the matching row to the new
   value.

The request body contains a single required field:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Required
     - Description
   * - ``ha_chassis_priority``
     - integer
     - yes
     - The new priority for this chassis in the router's
       ``HA_Chassis_Group``.

Changing a chassis priority may alter the active/backup roles. For
example, raising a backup chassis priority above the current primary
will make it the new active gateway, and OVN will trigger a failover.


DELETE /v2.0/agents/{agent_id}/l3-routers/{router_id} — Remove a router from a chassis
--------------------------------------------------------------------------------------

This operation removes a chassis from the ``HA_Chassis_Group`` of the
router's gateway ``Logical_Router_Port``. The implementation will:

#. Validate that the agent is associated with the router's
   ``HA_Chassis_Group``.
#. Delete the ``HA_Chassis`` row for this chassis from the
   ``HA_Chassis_Group``.

If the removed chassis was the **primary** (highest priority), OVN will
automatically fail over the gateway ``Logical_Router_Port`` to the next
highest-priority chassis.

If the last chassis is removed from the ``HA_Chassis_Group``, the
router's gateway port becomes unhosted. The automatic OVN L3 scheduler
(triggered by chassis events) will attempt to reschedule it if
eligible candidates are available.

This API call does not reschedule the current chassis priorities; this
operation, if required or needed, must be done by the user.


New API extension: ``l3-agent-scheduler-ha-priority``
-----------------------------------------------------

The current L3 Agent Scheduler API [3]_ response for
``GET /v2.0/routers/{router_id}/l3-agents`` returns a flat list of agents
with no indication of relative priority or failover order. In the classic
L3 agent model, this was not needed because each router was bound to
exactly one L3 agent (or one active + one standby for L3 HA).

In OVN, each gateway ``Logical_Router_Port`` can have up to
``MAX_GW_CHASSIS`` (currently 5) ``HA_Chassis`` entries [4]_, each with
a distinct integer priority (1 = lowest, N = highest, where N ≤
``MAX_GW_CHASSIS``). The chassis with the highest priority is the
**active** gateway; the others are ranked backups.

This information is critical for operators to understand the HA posture
of their routers. A new API extension will add the following field to the
agent objects returned by ``GET /v2.0/routers/{router_id}/l3-agents``:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ``ha_chassis_priority``
     - integer
     - The priority of this agent/chassis in the router's
       ``HA_Chassis_Group``. The highest value indicates the current
       primary (active) gateway chassis. Lower values indicate ranked
       backup chassis. If the extension is not loaded, this field is
       absent.

Example response with the extension enabled::

  {
    "agents": [
      {
        "id": "a1b2c3d4-...",
        "agent_type": "OVN Controller agent",
        "binary": "ovn-controller",
        "host": "gateway-node-1",
        "alive": true,
        "admin_state_up": true,
        "ha_chassis_priority": 5,
        ...
      },
      {
        "id": "e5f6a7b8-...",
        "agent_type": "OVN Controller agent",
        "binary": "ovn-controller",
        "host": "gateway-node-2",
        "alive": true,
        "admin_state_up": true,
        "ha_chassis_priority": 4,
        ...
      },
      {
        "id": "c9d0e1f2-...",
        "agent_type": "OVN Controller agent",
        "binary": "ovn-controller",
        "host": "gateway-node-3",
        "alive": true,
        "admin_state_up": true,
        "ha_chassis_priority": 3,
        ...
      }
    ]
  }

In this example, ``gateway-node-1`` is the primary (active) gateway
chassis (priority 5), and the other two are ranked backups.


Multiple gateway ports (multi-homing)
-------------------------------------

Since [5]_, Neutron supports routers with multiple gateway ports.
However, the current implementation creates a **single**
``HA_Chassis_Group`` **per** ``Logical_Router`` (named after the
logical router), not one per gateway ``Logical_Router_Port``. All
gateway LRPs on the same router share the same ``HA_Chassis_Group``
and therefore the same set of chassis with the same priorities.

The multi-homing anti-affinity is achieved at the **scheduler
selection** level: the ``OVNGatewayLeastLoadedScheduler`` receives the
``target_lrouter`` and penalizes chassis that already host other LRPs
for the same router, making it less likely that two gateway ports end
up with the same primary chassis. But the resulting chassis list is
written to the shared ``HA_Chassis_Group``.

Because of this one-HCG-per-router model, the L3 Agent Scheduler API
mapping is straightforward: one router maps to one
``HA_Chassis_Group``, which maps to one set of chassis/priorities.
The ``GET /v2.0/routers/{router_id}/l3-agents`` response returns one
agent entry per chassis in the group, regardless of how many gateway
``Logical_Router_Port`` rows the router has.


Interaction with the automatic OVN L3 scheduler
-----------------------------------------------

The automatic OVN L3 scheduler [6]_ (``OVNGatewayChanceScheduler`` or
``OVNGatewayLeastLoadedScheduler``) continues to operate independently.
It is triggered by ``Chassis`` events [7]_ (add, remove, bridge-mapping
changes, ``enable-chassis-as-gw`` changes) and fills
``HA_Chassis_Group`` slots up to ``MAX_GW_CHASSIS``.

When an operator manually schedules or removes a chassis via the API,
the resulting ``HA_Chassis_Group`` state is authoritative. The automatic
scheduler will **not** undo manual changes as long as the group already
has the maximum number of valid chassis. If a manual removal creates a
"hole" (fewer than ``MAX_GW_CHASSIS`` valid chassis), the automatic
scheduler may fill it on the next chassis event, consistent with its
existing behaviour for unhosted gateways.


OVN Northbound database loss
----------------------------

.. warning::

   If the OVN Northbound database is lost, the Neutron OVN sync tool
   (``neutron-ovn-db-sync-util``) will re-create the NB database contents
   from the Neutron database. However, the chassis scheduling state —
   that is, the ``HA_Chassis_Group`` and ``HA_Chassis`` rows that record
   which chassis host each router and at which priority — is **not**
   stored in the Neutron database. It lives exclusively in the OVN
   Northbound DB.

   As a consequence, any manually defined chassis-to-router assignments
   and their priorities will be **permanently lost**. After the sync tool
   recreates the ``Logical_Router_Port`` rows, the automatic OVN L3
   scheduler will build **new** ``HA_Chassis_Group`` entries from
   scratch, selecting chassis according to its configured algorithm
   (``OVNGatewayChanceScheduler`` or ``OVNGatewayLeastLoadedScheduler``).
   The resulting scheduling may differ significantly from the original:
   primary/backup roles may be reassigned, and any operator-customized
   priorities will not be preserved.

   Operators who rely on manual scheduling via this API should maintain
   an external record of their chassis assignments and priorities so that
   they can be re-applied after a full Northbound database recovery.


Implementation approach
-----------------------

* **OVN L3 plugin override:** The ``OVNL3Plugin`` in
  ``neutron/services/ovn_l3/plugin.py`` will implement
  ``list_l3_agents_hosting_router()``,
  ``list_routers_on_l3_agent()``,
  ``add_router_to_l3_agent()``, and
  ``remove_router_from_l3_agent()``
  by operating on the OVN NB IDL (``HA_Chassis_Group`` /
  ``HA_Chassis`` rows) instead of the ``RouterL3AgentBinding`` table.

* **OVSDB commands:** New or extended OVSDB commands in
  ``neutron/plugins/ml2/drivers/ovn/mech_driver/ovsdb/commands.py``
  will handle inserting and removing ``HA_Chassis`` rows with
  appropriate priority renumbering, reusing the existing
  ``_sync_ha_chassis_group`` pattern.

* **Extension loading:** The ``l3_agent_scheduler`` extension alias will
  be advertised when ML2/OVN is the active mechanism driver, making the
  endpoints available.

* **New extension definition:** A new API extension
  ``l3-agent-scheduler-ha-priority`` will be defined in neutron-lib
  (module ``l3_agent_scheduler_ha_priority``).
  It depends on ``l3_agent_scheduler`` and adds the
  ``ha_chassis_priority`` attribute to the agent response (GET) and
  the scheduling request body (POST).

* **Policy:** The existing policy rules will be reused:
  ``get_l3-agents`` for listing, ``create_l3-router`` for scheduling,
  and ``delete_l3-router`` for unscheduling. All are admin-only by
  default. No new policy rules are required.


What is out of scope for the initial version
--------------------------------------------

* Bulk scheduling or bulk priority reassignment in a single API call.
* A dedicated rebalancing API — rebalancing remains operator-driven via
  existing CLI tools (``neutron-ovn-db-sync-util``).
* Changes to the OVN L3 scheduler algorithms themselves.


References
==========

.. [1] HA_Chassis_Group migration for L3 scheduling:
   https://review.opendev.org/q/topic:%22bug/2092271%22

.. [2] OVN NB schema — HA_Chassis.priority limits (0–32767):
   https://github.com/ovn-org/ovn/blob/fed2f796895c8bcc04feec862632074350645bb9/ovn-nb.ovsschema#L806-L807

.. [3] L3 Agent Scheduler API reference:
   https://docs.openstack.org/api-ref/network/v2/index.html#l3-agent-scheduler

.. [4] OVN NB schema — HA_Chassis_Group, HA_Chassis, Logical_Router_Port:
   http://www.openvswitch.org/support/dist-docs/ovn-nb.5.txt

.. [5] Active-active L3 gateway with multi-homing spec:
   https://review.opendev.org/c/openstack/neutron-specs/+/870030

.. [6] OVN L3 Scheduler admin documentation:
   https://docs.openstack.org/neutron/latest/admin/ovn/l3_scheduler.html

.. [7] L3 HA Scheduling of Gateway Chassis (contributor docs):
   https://docs.openstack.org/neutron/latest/contributor/internals/ovn/l3_ha_rescheduling.html
