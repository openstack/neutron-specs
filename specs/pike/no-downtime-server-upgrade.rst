..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Upgrade controllers with no API downtime
========================================

https://blueprints.launchpad.net/neutron/+spec/online-upgrades

Problem Description
===================

Currently, database migration for a new major neutron release means full
shutdown of all neutron-server instances before `contract
<http://docs.openstack.org/developer/neutron/devref/alembic_migrations.html#expand-and-contract-scripts>`_
alembic migration scripts are applied.

When running a popular cloud, it's usually hard or impractical to shut down all
neutron-server instances before applying all ``contract`` alembic migration
scripts that may take a while, especially for databases with a lot of data to
migrate. Such a shutdown requires a significantly more elaborate planning,
trying to squeeze the upgrade process in a maintenance window with less
disruption to API users. For clouds with high SLAs, it may be impossible to
shutdown all Neutron API endpoints at one time, leaving users with no
Networking API for extended time. Due to dependencies from other services (i.e.
Nova) on Neutron API availability, upgrades have a greater impact as Neutron is
a central service in any Openstack installation.

This spec describes an approach to allow for non-impacting neutron-server
upgrades, leaving instances running in a cluster. Instead of full shutdown,
operators will be able to upgrade the services in rolling mode, upgrading each
node running the service without disruption for other such nodes. If Networking
API is served by multiple nodes hidden behind a load balancer, that approach
should allow for no-downtime upgrade experience. Ideally, users would not
notice any issues accessing Neutron API services for the entirety of the
upgrade.

.. note::

   Running mixed major versions of neutron-server in a cloud opens the question
   of how to mitigate slight differences of API behaviour between those
   versions.  For example, in Newton, all resources received a new
   ``project_id`` field.  If we would run mixed Mitaka/Newton versions of
   neutron-server behind a round robin load balancer, then consequent GET calls
   for the same resource would result in different reply payloads, depending on
   which particular neutron-server is hit by an API request.

   Enforcing consistent API behaviour for mixed version environment, or pinning
   API behaviour to a version that would describe previous major version
   behaviour, is *out of scope* for the proposal. Consequent blueprints may
   clarify best practices, or propose mechanisms for strict API behaviour
   control.

Proposed Change
===============

Since neutron-server downtime derives solely from executing unsafe ``contract``
upgrade migrations while neutron-server is operational, the solution is to make
those migrations safe for online execution, or eliminate them. This is achieved
with two major changes:

#. Time consuming data migration changes are moved from neutron-db-manage phase
   into neutron-server itself, so that data is migrated while the service is up
   and serving requests, instead of while it's fully shut down. Data migration
   process will be ``lazy``, happening at the time when a resource is touched
   by plugin code. That said, users should not be able to switch to a next
   major version before they complete migration for all remaining resources. We
   will provide a tool to trigger pending migrations (preferrably in chunks)
   that will become part of preparation process for next upgrade. The tool will
   be modeled similar to ``online-data-migrations`` command found in Cinder.

#. Remaining schema contraction changes are postponed to the time when:

   - all the data is migrated from old tables/columns, and
   - no neutron-server instances running in a cluster are able to access
     obsolete tables/columns.

.. note::

   This idea is not new, other projects already got rid of unsafe migration
   scripts that would require offline execution. Among those projects are Nova
   and Cinder.

Data migration between multiple tables/columns implemented in neutron-server
runtime is potentially error prone and requires specific reviewer attention. It
would be impractical to expect proper attention to those intricacies for any
patch that needs to read or update a database model. So the first step is to
isolate the layer that has access to database models behind a special facade.
The base of the facade is oslo.versionedobjects and corresponding NeutronObject
framework that is already in tree and is successfully used by several features
(qos, vlan-aware-vms). The work to switch all the plugin code that accesses
database using SQLAlchemy models, to object facade, is ongoing and is tracked
as a `separate blueprint
<https://blueprints.launchpad.net/neutron/+spec/adopt-oslo-versioned-objects-for-db>`_.

Once plugin code is switched to using objects for resource persistence, we can
implement any needed *data* migration rules in a single place, in a
corresponding object class, isolating consuming code from all the complexities
of the migration/conversion process.

Since in-object translation of data is lazy, at the point of next major upgrade
some resources may still have their database records not transitioned to a new
format. To facilitate next major upgrade, `a new neutron-db-manage CLI command
(migrate-data) <https://review.openstack.org/#/c/432494/>`_ will be implemented
that can be executed by operators before proceeding with the upgrade. The
command will migrate any remaining data models to a new format. The command
will run data migration scripts exposed via a stevedore namespace, for external
subprojects to be able to make use of it.

Even with persistence facade, some work on migration mechanism and techniques
is expected. For the start, a Pike feature that would need a schema/data
migration change will be identified and used as a ``guinea pig`` to explore the
proposal practicalities.

At the time of writing, `port bindings rework
<https://bugs.launchpad.net/bugs/1580880>`_ is probably the best candidate to
try the new approach.

As for unsafe *schema* changes, if at release X we want to introduce a contract
migration we can only do so destructively in release X+2 (i.e. after all the
data used by X and X+1 located in the old schema is migrated), which guarantees
that whenever the deployer upgrades to X+2 (from X+1), the server instance
running behind won't hit the data/code being affected by the schema migration.
At this point, even seemingly unsafe operations like dropping tables or columns
become safe to execute while neutron-server instances are online. For Ocata and
later, there should be no new ``contract`` alembic scripts at all. Those may
show up again in later releases.

To achieve the goal for Newton to Ocata and later upgrades, we should guarantee
that all patches will follow decided path during Ocata and later. This will be
achieved by both automation as well as social means.

For the former, we introduce a `functional test
<https://review.openstack.org/#/c/400239/>`_ that fails on attempt to execute
an operation known to be unsafe.

We will probably not be able to catch everything programmatically, so we still
need to make sure core reviewer team is aware of new requirements, and
proactively track all proposed alembic migrations at least first cycles until
it becomes a habit of an average Joe the Reviewer to spot and -1 unsafe
patches.

To make sure the new upgrade mode works, a new gating grenade job will be
implemented that will run multiple neutron-server instances of different major
versions. Access to Networking API will be implemented using a lightweight load
balancer (``haproxy``) hiding multiple instances of neutron-server behind it.
We may also consider moving known consumers of Neutron API (for example, Nova)
to the 'old' subnode to make sure it can talk to newer as well as older Neutron
in the same setup.  Only two adjacent major versions of neutron-server will be
used in the grenade job to conform to `assert:supports-upgrade tag requirements
<https://governance.openstack.org/reference/tags/assert_supports-upgrade.html#requirements>`_.

Action items
------------

(The feature assumes completion of `adopt-oslo-versioned-objects-for-db
blueprint
<https://blueprints.launchpad.net/neutron/+spec/adopt-oslo-versioned-objects-for-db>`_
but does not strictly depend on it.)

#. Block unsafe contract migrations at the start of Ocata (*Done*).
#. Explore practicalities of the proposal for data migrations for a new feature.
#. Add a voting grenade job running different major versions of neutron-server.
#. Document new upgrade path in ops upgrades guide, with its limitations.
#. Update devref and wider audience about new requirements.

References
==========

None.
