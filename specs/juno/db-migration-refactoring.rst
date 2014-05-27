..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Database Migration Refactoring
==========================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/db-migration-refactor

The neutron database schema will be made consistent. It will be independent of
the configured core plugin and service plugins.


Problem description
===================

The database schema is different based on which core and service plugins are
configured. If a new or different plugin is configured after a deployment is
started, the earlier migrations that correspond with the new configuration will
be missing.

This means that the migration process is not idempotent, resulting in schema
versions which depend on the current configuration. In other words, when one
environment's schema is at a specific version it may not be the same as another
environment at the same version. This even happens in the same environment
during a downgrade if the configuration changes.


Proposed change
===============

Make all migrations unconditional. Make a single point in the migration
timeline where all tables are created and the database schema is the same no
matter which plugins are used.

This solution will introduce a "healing" migration that will call the DDLs
needed to complete the database schema so that it contains all tables for all
database models. This migration will occur in the timeline between the
**icehouse_release** version and the **juno_release** version.

The refactoring will comprise the following:

1) Investigation of models and current migrations to establish what is needed
   to achieve the complete database schema.

2) A healing migration that ensures that the database schema is complete and
   consistent. The version will be named **db_healing**.

3) All migrations after **db_healing** will be unconditional. This requires
   updating the behavior of the --autogenerate for neutron-db-manage, and
   documentation for developers.

The healing migration will work like any other alembic migration, i.e. it will
allow upgrade and downgrade. However, it will be different in the following
aspects:

* In the upgrade direction the healing migration will introspect the schema and
  only add tables that are missing. Therefore it cannot be run in offline
  mode. (See also :ref:`onlinedetails`.)

  Upgrade Notes:

  * When the healing migration starts, all table schema changes for existing
    tables will already be in place (either directly from the model, or from
    previous migrations). For each missing table, the healing migration will
    add the table by creating its schema from the model.

  * The healing migration may need to alter some existing tables to match its
    model. We won't know for sure until the proposed implementation is tested
    in more detail. These alterations, if needed, will be dealt with on a
    case-by-case basis. For more details, see the :ref:`implementation`
    section.

* In the downgrade direction the healing migration will make no schema
  changes. This means it will not remove any tables.

  Downgrade Notes:

  * If a deployment is upgraded and then downgraded through the healing
    migration, the schema will contain tables that were not present before. In
    other words, if a deployment downgrades back to Icehouse or Havana, Neutron
    will appear and behave as before, yet if you look at the DB it is not the
    old Icehouse or Havana DB but a healed one.

  * If during the upgrade a table was altered to match its model, then the
    downgrade will reverse the alteration if needed.


Migration Timeline
------------------

.. blockdiag::

  blockdiag timeline {
    orientation = portrait;
    default_group_color = lightgreen;

    '...' -> icehouse_release -> '....' -> db_healing;
    db_healing -> '.....' -> juno_release -> '......';
    group {
      db_healing;
    }
  }


.. _onlinedetails:

Online Requirement
------------------

More reasons why we cannot support offline mode for the healing migration:

1) We cannot use "create table if not exists" because it is only supported by
   specific sql dialects.

2) Dependencies between tables (e.g. foreign keys) mean we would need to check
   that all tables with dependencies on the current one are already created.

3) Alembic has no support for conditional DDL.


Alternatives
------------

Instead of a healing migration we could break the migration timeline after
Icehouse and create a new one beginning at Juno that includes all tables. A
manual script could be provided to convert the schema from the old timeline to
the new. This alternative approach has the following disadvantages:

* It would not allow a downgrade via migration.

* Switching from the old timeline to the new timeline is more a complex process
  for the deployer and DB Administrator than a simple migration in one timeline.

* We would need to support two migration timelines in neutron until the
  Icehouse release is deprecated.


Data model impact
-----------------

Some database models may need to be updated in order to have non-conflicting
models based on core and service plugins.


REST API impact
---------------

None


Security impact
---------------

None


Notifications impact
--------------------

None


Other end user impact
---------------------

End (non-admin) users should see no impact.


Performance Impact
------------------

None

Other deployer impact
---------------------

The healing migration is an online operation that must be run by the DB
Administrator. Thus the deployer must co-ordinate with the DBA when upgrading
or downgrading through Juno.

The healing migration will be run when migrating from Icehouse or earlier
to Juno. It behaves like a normal migration, but it does not support upgrade in
offline mode.

Downgrade of the healing migration does nothing. Thus all tables are present in
the schema if downgrading from **db_healing** to a previous version.

Note: Greenfield deployment of the Juno release or later will start at the new
migration timeline and therefore no healing will be involved.


Developer impact
----------------

None


.. _implementation:

Implementation
==============

Most of the work lies in developing a robust healing migration. Alembic will be
used where possible to maximize automation, but some healing may need to be
manually coded.

Examples of conflicts which may need manual coding to resolve:

* If two different migrations for two different plugins add an attribute with
  the same name but different type.

* If an ENUM was modified in an earlier migration but its specification was not
  updated for PostgreSQL.


Assignee(s)
-----------

* akamyshnikova
* libosvar
* rpodolyaka


Work Items
----------


Dependencies
============


Testing
=======

Unit (and functional?) testing of migrations shall be added. We plan to utilize
the unit test framework from the graduated oslo.db package.


Documentation Impact
====================

* Create Release Note for this change.

* Update Operators Guide for upgrading.

* Update Developer Documentation for creating migration scripts.


References
==========

