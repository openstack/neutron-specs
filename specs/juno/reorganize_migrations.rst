..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================================
Reorganize neutron migrations (Once they've been 'healed')
===========================================================

https://blueprints.launchpad.net/neutron/+spec/reorganize-migrations

A good chunk of current migrations apply only to certain plugins making them
configuration-dependent. A migration path dependent on configuration poses a
serious upgrade problem. The Neutron DB is being restructured in a way that
the DB schema will always be the same regardless of Neutron's configuration.
For this reason, logic for handling configuration-dependent migrations should
go; similarly configuration dependent migrations should be made 'independent'.
Possibly they should also be squashed so that by reducing the overall number
of migration we might be able to speed up DB ugrade process.

The work described in this blueprint relies on the Neutron DB healing
blueprint, whose spec can be found at: https://review.openstack.org/95738/

Problem description
===================

There are currently over 100 migrations in Neutron's DB upgrade path.
A migration often applies only to certain plugins, and is skipped if currently
neutron is configured with a different plugin. This creates a serious upgrade
problem.

As an example, the devstack job for Neutron for testing an upgrade from havana
to icehouse fails because the 'metering' service plugin is specified only in
the icehouse configuration; however since it was introduced in havana, some
fundamental migrations to make it work where skipped. Hence the upgrade fails
as soon as the first migration touching metering DB models is encountered.

This problem is already being dealt with by "healing" neutron DB schema in a
way such that its end state will be the same regardless of conf values.

Nevertheless, there a few more things to take care of:

* Existing DB migrations which apply to specific plugins must become
  plugin-agnostic

* The logic for handling plugin-specific migrations should go in order to
  prevent plugin-specific migration from being pushed in the future.


Proposed change
===============

The proposed change is rather simple: remove the logic for skipping
migrations according to the current configuration, and make all migrations
"global".

As this blueprint looks back at the whole migration history, which stretches
back to folsom, this is also a good chance to coalesce this rather long path
by squashing intra-release migrations together, and removing from the path
versions which are now not anymore supported.

For instance, the revised path could:
1) Start at Havana Release

2) Have a single, configuration-independent, migration to Icehouse

3) Resume the 'traditional' path from Icehoue up to trunk, ensuring that
all migrations in this path become however configuration-independent

This new migration path MUST ensure correct operation of upgrade/downgrades
from Havana to Icehouse. To this aim, this change will introduce:

* A new 'Havana' initial state. This will be independent from configuration

* A single migration to go from Havana to Icehouse and vice versa.
  This migration will be able to upgrade an Havana database to Icehouse and
  viceversa. The downgrade side of this migration will assume the Icehouse
  schema is already in an "healed" state. Hence the following step.

* A healing step immediately before Icehouse. This migration will
  have only the downgrade side. This is for ensuring the database is in a
  healed state as the new downgrade migration to Havana makes this
  assumption. Being the healing migration idempotent, this should not
  be a problem.

NOTE: If there is consensus to keep allowing users running unsupported
releases to upgrade to Icehouse directly then initial statuses for Folsom
and Grizzly might be kept.

Alternatives
------------

One alternative would be to leave everything as it is at the moment, and add
a test (integrated with flake8) to prevent new plugin-dependent migrations
from being added to the path; Also the check for asessing whether a migration
should be executed or skipped will be bypassed.

This should work, but it will still leave us with a lot of plugin-specific
migrations which might confuse users. Also, there will be leftover, unused
code in the source tree which will eventually end up bitrotting.

Data model impact
-----------------

The migration path up to Havana will be removed.
This can be substituted with Grizzly if there is an agreement to provide
users running the now unsupported Grizzly to uprade to a more recent release.

Intra-release migrations between Havana and Icehouse (and possibly between
Grizzly and Havana) will be squashed into a single migration.

REST API impact
---------------

None

Security impact
---------------

Nada

Notifications impact
--------------------

Nonw

Other end user impact
---------------------

None

Performance Impact
------------------

If migrations are squashed, upgrade will obviously be faster.
No impact for runtime performance.

Other deployer impact
---------------------

Operators which already already running havana should be able to upgrade
to icehouse without any problem.

For operators running Icehouse, Havana downgrade should be smooth, both if
they executed or not already the 'healing' migration.

Operators running versions prior to Havana might still be supported,
if versions older than Havana are kept in the migration path.

Developer impact
----------------

It will be no longer possible to create plugin-specific migrations.

Implementation
==============

In rough terms we might expect a first step where the migration logic is
removed, and all migrations made independent of configuration.
In a second step instead a new initial state will be defined and intra-release
migrations prior to Icehouse will be squashed.

Assignee(s)
-----------

salvatore-orlando
akamyshnikova

Work Items
----------

Even if these work items are not orthogonal, dependencies among them are not
necessarily represented by the order in which appear in this document.

* remove all folsom and grizzly migrations. Create a new "Havana" initial
  migration - in a "healed" state.
* make conditional migrations unconditional
* ensure a healing step is performed also in downgrade direction.
  This is for ensuring unconditional downgrades don't fail because of a
  database not in a "healed" state.
* remove logic for managing conditional migrations
* squash migration between havana and icehouse in a single migration
* deal with conditional migrations between "icehouse" and the healing step


Dependencies
============

DB migration refactoring is a prerequisite for this blueprint.
https://review.openstack.org/#/c/95738/

Testing
=======

No additional tests should be added. Instead, some unit tests concerning
plugin-specific migrations might be removed.

Existing grenade tests should be enough to validate the new upgrade path.

Documentation Impact
====================

None

References
==========

None
