..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Per-network metadata enable/disable for ML2/OVN
===============================================

https://bugs.launchpad.net/neutron/+bug/2169598

Currently the OVN mechanism driver controls metadata service creation via a
single global configuration option ``ovn_metadata_enabled`` (in the ``[ovn]``
section of ``neutron.conf``).  When enabled, a metadata port
(``device_owner=network:distributed``) is automatically created for every
network that has DHCP-enabled subnets.  There is no per-network control over
this behaviour.

This spec proposes exposing a per-network ``metadata_enabled`` attribute that
lets users enable or disable the metadata service on individual networks,
superseding the global default for any network where the attribute is
explicitly set.


Problem Description
===================

Two operational issues motivate this feature:

#. **Accidental metadata port deletion breaks cloud-init.**
   If a user or admin accidentally deletes the metadata port
   (``device_owner=network:distributed``), it is not recreated until a subnet
   update triggers the reconciliation path.  Metadata (and therefore
   cloud-init) stops working for every VM on that network until the port is
   restored.

#. **Unnecessary metadata ports waste resources.**
   Some networks do not need metadata at all — for example, provider networks
   used purely for external connectivity, or networks whose VMs are configured
   without cloud-init.  When ``ovn_metadata_enabled`` is ``True``, metadata
   ports are forced onto every network regardless.

With a per-network attribute, operators and tenants gain:

* **Fine-grained control** — enable metadata only where it is needed.
* **Self-service recovery** — if a metadata port is accidentally deleted, the
  user can toggle ``metadata_enabled`` to ``False`` and back to ``True`` (or
  simply set it to ``True``) to recreate the port without admin intervention.
* **Resource efficiency** — networks that do not need metadata avoid
  unnecessary ports and metadata-proxy processes.


Proposed Change
===============

Overview
--------

Introduce a new API extension ``network-metadata`` that adds a Boolean
attribute ``metadata_enabled`` to the ``network`` resource.  The attribute is
stored in a new database table and processed by the ML2/OVN mechanism driver
to create or delete the metadata port on each network accordingly.  The OVN
agent metadata extension is enhanced to react to per-network metadata changes
originating from the Neutron database or the OVN Southbound database.


REST API Changes
----------------

A new API extension ``network-metadata`` adds the ``metadata_enabled``
attribute to networks.

**Extension definition** (in ``neutron-lib``):

* ``ALIAS``: ``network-metadata``
* ``NAME``: ``Network Metadata``
* ``DESCRIPTION``: Per-network metadata service enable/disable.
* Parent extension: ``network``

Attribute definition::

    METADATA_ENABLED = 'metadata_enabled'

    RESOURCE_ATTRIBUTE_MAP = {
        'networks': {
            METADATA_ENABLED: {
                'allow_post': True,
                'allow_put': True,
                'is_visible': True,
                'default': None,       # None means "use global default"
                'convert_to': convert_to_boolean_if_not_none,
                'is_filter': True,
                'is_sort_key': True,
            },
        },
    }

Semantics of the ``metadata_enabled`` attribute:

* ``None`` (default) — inherit the global ``ovn_metadata_enabled`` value.
  This is the behaviour for all existing networks after the upgrade and for
  new networks where the attribute is not specified.
* ``True`` — metadata is explicitly enabled on this network.  The metadata
  port is created if it does not exist.
* ``False`` — metadata is explicitly disabled on this network.  Any existing
  metadata port is deleted.

The effective metadata state for a network is computed as:

.. code-block:: python

    def is_metadata_enabled(network):
        per_network = network.get('metadata_enabled')
        if per_network is not None:
            return per_network
        return cfg.CONF.ovn.ovn_metadata_enabled

**Create network** — ``POST /v2.0/networks``:

.. code-block:: json

    {
        "network": {
            "name": "my-network",
            "metadata_enabled": true
        }
    }

**Update network** — ``PUT /v2.0/networks/{network_id}``:

.. code-block:: json

    {
        "network": {
            "metadata_enabled": false
        }
    }

To reset to the global default:

.. code-block:: json

    {
        "network": {
            "metadata_enabled": null
        }
    }

**Show / List** — the attribute appears in the response:

.. code-block:: json

    {
        "network": {
            "id": "...",
            "name": "my-network",
            "metadata_enabled": true,
            "..."
        }
    }

CLI examples (via ``openstackclient``):

.. code-block:: text

    # Create with metadata explicitly enabled
    openstack network create --metadata-enabled my-network

    # Create with metadata explicitly disabled
    openstack network create --no-metadata-enabled my-network

    # Update to disable metadata
    openstack network set --no-metadata-enabled my-network

    # Update to re-enable metadata (recreates the port if missing)
    openstack network set --metadata-enabled my-network

    # Show metadata status
    openstack network show my-network -c metadata_enabled


Data Model Changes
------------------

A new table ``networkmetadata`` is added:

.. code-block:: sql

    CREATE TABLE networkmetadata (
        network_id VARCHAR(36) NOT NULL,
        metadata_enabled BOOLEAN,
        PRIMARY KEY (network_id),
        FOREIGN KEY (network_id) REFERENCES networks(id) ON DELETE CASCADE
    );

The SQLAlchemy model (in ``neutron/db/models/network_metadata.py``):

.. code-block:: python

    class NetworkMetadata(model_base.BASEV2):
        __tablename__ = 'networkmetadata'

        network_id = sa.Column(
            sa.String(36),
            sa.ForeignKey('networks.id', ondelete='CASCADE'),
            primary_key=True)

        metadata_enabled = sa.Column(sa.Boolean(), nullable=True)

        network = orm.relationship(
            models_v2.Network,
            load_on_pending=True,
            backref=orm.backref(
                'metadata',
                lazy='joined',
                uselist=False,
                cascade='delete'))

        revises_on_change = ('network', )

The ``nullable=True`` column allows storing ``None`` to represent "use global
default".  An Alembic migration adds the table.

.. note::

   An alternative to a separate table is adding the column directly to the
   ``networks`` table (or using ``standardattrs``).  A separate table is
   preferred for consistency with other network extension attributes (e.g.
   ``networkpvlan``, ``networkdnsdomain``) and to avoid schema changes to the
   core ``networks`` table.


ML2/OVN Mechanism Driver Changes
---------------------------------

The ML2/OVN mechanism driver is extended to honour the per-network attribute:

#. **``create_network_postcommit``** — after network creation, if metadata is
   effectively enabled (per-network attribute or global default), the driver
   proceeds as today to create the metadata port.  If metadata is effectively
   disabled, the metadata port creation is skipped.

#. **``update_network_postcommit``** — when ``metadata_enabled`` changes:

   * ``False → True`` (or ``None → True`` when global is ``False``):
     Call ``create_metadata_port()`` for the network.
   * ``True → False`` (or ``None → False`` when global is ``True``):
     Find and delete the existing metadata port
     (``device_owner=network:distributed``).

#. **``OVNClient.create_metadata_port``** — the existing guard:

   .. code-block:: python

       if not ovn_conf.is_ovn_metadata_enabled():
           return

   is replaced with a check that considers the per-network attribute:

   .. code-block:: python

       if not is_metadata_enabled_for_network(network):
           return

   where ``is_metadata_enabled_for_network()`` resolves the effective state as
   described above.

#. **``OVNClient.update_metadata_port``** — same guard replacement.

#. **``OVNClient.delete_metadata_port``** — new helper that finds and deletes
   the metadata port for a given network (and segment, for routed networks).

#. **DB sync** (``ovn_db_sync.py``) — the sync logic is updated so that:

   * Networks with ``metadata_enabled=True`` (or ``None`` when the global is
     ``True``) that lack a metadata port get one created.
   * Networks with ``metadata_enabled=False`` that still have a metadata port
     get it deleted.

This ensures that the system converges to the desired state during periodic
DB synchronisation.


OVN Agent Metadata Extension Changes
-------------------------------------

The OVN agent metadata extension
(``neutron/agent/ovn/extensions/metadata.py``) currently manages metadata
proxy processes for every network with bound ports on the local chassis.

The following changes are needed:

#. **Filtering by ``metadata_enabled``** — when the agent builds the list of
   networks for which to spawn ``haproxy`` metadata-proxy processes, it must
   check the per-network ``metadata_enabled`` attribute.  Networks that have
   ``metadata_enabled=False`` (and networks with ``metadata_enabled=None``
   when the global is ``False``) must be excluded.

   The metadata port already encodes whether metadata is desired: if the port
   does not exist, the agent has nothing to bind.  However, as an explicit
   safety measure, the agent can also read the ``metadata_enabled`` flag from
   the OVN Northbound logical switch ``external_ids`` (see below) to avoid
   spawning a proxy for a network where metadata was explicitly disabled.

#. **Propagating the attribute to OVN** — the ML2/OVN driver sets an
   ``external_id`` on the OVN logical switch to communicate the per-network
   metadata state to the agent:

   .. code-block:: text

       external_ids:neutron:metadata_enabled=True|False

   When the attribute is ``None`` (inherit global), the key is omitted, and
   the agent falls back to the global ``ovn_metadata_enabled`` config.

#. **Reacting to changes** — the agent already monitors
   ``Port_Binding`` events.  When a metadata port is created or deleted as a
   result of toggling ``metadata_enabled``, the agent reacts naturally:

   * Metadata port created → ``PortBindingCreateWithChassis`` fires →
     agent provisions the metadata namespace and starts the proxy.
   * Metadata port deleted → ``PortBindingDeletedEvent`` fires →
     agent tears down the metadata namespace and stops the proxy.

   No new SB events are required; the existing event handlers are sufficient.

#. **Sync path** — the ``sync()`` method is updated to respect the
   per-network attribute.  During full resync the agent should tear down
   metadata proxy processes for networks where metadata has been disabled and
   create them for networks where metadata has been enabled.


Upgrade Impact
--------------

* The Alembic migration adds the ``networkmetadata`` table.  On upgrade, the
  table is empty, so all networks inherit the global ``ovn_metadata_enabled``
  value — this preserves the existing behaviour with zero disruption.

* Operators can then selectively set ``metadata_enabled`` on individual
  networks to override the global default.

* No changes are required to existing configuration files.

* Rolling upgrades are supported: old agents that do not understand the
  attribute continue to use the global flag.  The per-network attribute only
  takes effect once both the API server and the agent are upgraded.


Scope and Limitations
---------------------

* This feature is **ML2/OVN only**.  Other mechanism drivers (ML2/OVS) are out
  of scope because this feature only applied to ML2/OVN networks.

* The ``metadata_enabled`` attribute is **not** tied to ``enable_dhcp`` on
  subnets.  They are independent: a network can have DHCP disabled but
  metadata enabled (the metadata port is still created with fixed IPs on
  DHCP-enabled subnets), or DHCP enabled but metadata disabled.

* Policy control: the default policy allows any network owner to set
  ``metadata_enabled``.  Operators can restrict this to admin-only via
  ``policy.yaml`` if desired.


References
==========

* Launchpad RFE: https://bugs.launchpad.net/neutron/+bug/2169598
* Bug #1764408: [OVN] DHCP ports don't recreated after deleting them,
  https://bugs.launchpad.net/neutron/+bug/1764408
* Bug #2015377: Recovery mechanism for missing metadata ports,
  https://bugs.launchpad.net/neutron/+bug/2015377
