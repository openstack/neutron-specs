..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Online Schema Migrations
========================

https://blueprints.launchpad.net/neutron/+spec/online-schema-migrations

:Author: Mike Bayer <mike.bayer@redhat.com>

This specification discusses the issue of database schema migrations which
may proceed while allowing both the previous and the updated version of the
Neutron database API to run against that schema at the same time.  This is
part of a larger approach which is to allow a Neutron application to
be upgraded to a new version without incurring downtime while the database
schema is migrated.

To achieve this goal fully, several areas must be addressed:

* The first is that database schema migrations can be applied which
  don't impact the old version of the software as it runs.  These migrations
  are referred to as **expansive** migrations, which only add new elements,
  never removing any.  Once the old software has been entirely replaced
  with the new version, a second series of migrations known as the **contract**
  migrations are run separately; these migrations remove the old elements
  that are no longer used.

* The newer version of the software must also be prepared to accommodate
  for the fact that the old version of the software may still be running
  as well, meaning it may need to read and persist data from both the old
  and new schema structures simultaneously.

* Within the scope of the new software referring to both versions of the
  schema, a strategy for data migrations must be devised.  These migrations
  can run over time as a function of the data access code itself slowly
  moving data to the new format, or can run as separate scripts or processes.

* The sequence of movements between expand/contract, old and new software
  versions, and migration of data must also be orchestrated fully.
  Front-end RPC clients and database access services are typically upgraded
  independently of each other, and additionally at which point "contract" is safe to run must
  be established.

This blueprint is primarily concerned with only the **first** bulletpoint,
that of organizing schema migrations such that those which are strictly
"expansive" may be run separately from those which are "contractual".
The other bullets above will need to be considered separately.

Note that an approach to the problem addressed here has already been
accepted for Nova, also called "online schema migrations".   The
specification here builds upon the work of Nova's, proposing
essentially the same concept, but implemented slightly differently,
in such a way that there is no
sharp break from Neutron's existing system of using Alembic migration
scripts, and does not abandon the use of version identifiers which
identify an explicit, known state of the schema.   There is also
a proposal for upstream changes in Alembic so that both Nova's "live"
approach and the "scripted" approach here can share the same codebase
against a revised Alembic autogeneration API that allows much greater
extensibility.


Problem Description
===================

Database migrations of Neutron and other Openstack applications traditionally
involve the replacement of some version of the schema with another one;
tables and columns are dropped, new ones added.   This change
necessarily involves that the software which communicates with the schema
must also change at the same time, where the old version is shut down
completely before beginning the migration, the migration then proceeds fully
offline, and then the new version is started.   In terms of a multiple-node
Openstack deployment, this means that the entire application
on all nodes must be fully shut down and upgraded globally all at once.
The offline migration may also be time consuming in terms of what kinds of
operations are present and what target database is in use.

The business requirements of many key Openstack consumers is such that
the downtime involved with fully upgrading all nodes simultaneously as
well as running full schema migrations during that downtime is no
longer acceptable; a new approach that allows the application to keep
running while the migration goes on must be developed, in particular
for key Openstack components such as Nova, Neutron, and Cinder.


Proposed Change
===============

Within this document we will address the goal of organizing schema
migrations into "expand" and "contract" phases that are also linked to
major release versions.   The phases are as follows:

* Migrations that run under "expand" are "additive" (e.g. tables,
  columns,  indexes and constraints are only created, not dropped)
  only, and are safe to run while the old version of the application
  continues to run.

* Migrations that run under "contract" are "subtractive" (e.g. tables,
  columns, indexes and constraints are only dropped, not created) and
  only run once software running on all nodes communicates exclusively
  with the new version of the schema, and all data has been migrated.

The steps involved in each of the two phases will be rendered as
explicit migration directives within Alembic migration scripts, as is
already the case for Neutron.  The only difference will be that a
given migration will be broken out into individual scripts for each
phase of operation that the migration includes.   These scripts will
be assembled into semi-independent lineages that can be run
separately.  These lineages will also be classified among release
versions, so that migration lineages will be targetable at the level
of both release and phase, e.g.  "expand liberty",  "contract M", etc.

The new scheme is supported by Alembic's  recently added support for
long-lived branches, roots, branch names, and individual file
directories.   The workflow can be implemented at a proof-of-concept
level without writing any new code,  by creating the new directory
structure and manually assembling new migration files into the
appropriate branches using Alembic's updated command line tools.

However, in order to facilitate the use of Alembic autogenerate, new
features will be added upstream to Alembic's autogenerate API which
will allow for the creation of custom autogenerate behaviors and
filesystem flows.  We will build a new tool that adapts Nova's current
online schema migrations logic to this new API such that the logic
used to group migrations into "expand"  and "contract" steps may now
stream those instructions into individual migration files, targeted
into the file structure referred to above.  It is  hoped that this
same tool will also be able to continue to send migration directives
directly to a database as well, thus allowing Nova's current "live"
approach to be rolled into the same codebase.  Improvements and
behavioral contracts for the "expand" / "contract" workflow system
will apply both to the "live" and "scripted" approaches, thus making
the two approaches that much more interchangeable.


Alembic Migrations
------------------

Right now, Neutron makes use of Alembic migrations, which involves a
series of migration scripts organized into the
``neutron/db/migration/alembic_migrations/versions`` directory.  These
scripts are organized into a kind of backwards linked list structure,
where each script is identified by a six-byte hash scheme, and
contains a variable that links it to the *previous* hash in the
series.   The rationale for this linked structure is that new versions
can be inserted into the middle of the chain without impacting more
than one existing migration file; by using a "backwards" linking
model, and new versions can be added to the end of the list without
impacting any existing versions.

Recent versions of Alembic have been enhanced to reconsider this
"backwards linked list" structure as just a specialization of a more flexible
structure, the directed acyclic graph, or DAG.   In this approach, we
remove the requirement that each migration script can only refer to a single
anscestor (e.g. dependency), as well as the requirement that only one
migration script can refer to a particular ancestor.   The structure basically
becomes open to the concepts of branching and merging which are very
familiar in version control systems.   Alembic now has the ability to
run upgrades or downgrades along individual branches which are tracked
individually within a database schema, meaning a schema's "head" version
may be in fact a series of hashes, each representing the "head" of an
individual revision stream.  The branches can optionally originate from entirely
independent root revisions with no dependencies on each other, and can
also be organized into individual subdirectories.   Revisions within
branches can also refer to specific revisions within other branches as
dependencies, and branches may be merged back together into a single
revision stream.

Expand and Contract Scripts
---------------------------

The current design of a migration script includes that it indicates
a specific "version" of the schema, and includes directives that apply
all necessary changes to the database at once.  If we look for example
at the script ``2d2a8a565438_hierarchical_binding.py``, we will see::

    # .../alembic_migrations/versions/2d2a8a565438_hierarchical_binding.py

    def upgrade():

        # .. inspection code ...

        op.create_table(
            'ml2_port_binding_levels',
            sa.Column('port_id', sa.String(length=36), nullable=False),
            sa.Column('host', sa.String(length=255), nullable=False),
            # ... more columns ...
        )

        for table in port_binding_tables:
            op.execute((
                "INSERT INTO ml2_port_binding_levels "
                "SELECT port_id, host, 0 AS level, driver, segment AS segment_id "
                "FROM %s "
                "WHERE host <> '' "
                "AND driver <> '';"
            ) % table)

        op.drop_constraint(fk_name_dvr[0], 'ml2_dvr_port_bindings', 'foreignkey')
        op.drop_column('ml2_dvr_port_bindings', 'cap_port_filter')
        op.drop_column('ml2_dvr_port_bindings', 'segment')
        op.drop_column('ml2_dvr_port_bindings', 'driver')

        # ... more DROP instructions ...

The above script contains directives that are both under the "expand"
and "contract" categories, as well as some data migrations.  the ``op.create_table``
directive is an "expand"; it may be run safely while the old version of the
application still runs, as the old code simply doesn't look for this table.
The ``op.drop_constraint`` and ``op.drop_column`` directives are
"contract" directives (the drop column moreso than the drop constraint); running
at least the ``op.drop_column`` directives means that the old version of the
application will fail, as it will attempt to access these columns which no longer
exist.

The data migrations in this script are adding new
rows to the newly added ``ml2_port_binding_levels`` table.   Data migrations
may or may not be "safe" to run within the "expand" or "contract" phase,
depending on the nature of the data.  It is expected that most data migrations
will run outside of migration scripts going forward, and instead be implemented
as part of the model/API layer as the application runs.

Note that this spec suggests, but not requires Neutron to move to live
data migrations implemented in the application instead of migration
scripts. This part will require a separate consideration and is out of
scope for the spec.

Under the proposed plan, the above script, assuming it were added as part
of the new architecture, would be stated as two scripts; an "expand" and a
"contract" script::

    # expansion operations
    # .../alembic_migrations/versions/liberty/expand/2bde560fc638_hierarchical_binding.py

    def upgrade():

        op.create_table(
            'ml2_port_binding_levels',
            sa.Column('port_id', sa.String(length=36), nullable=False),
            sa.Column('host', sa.String(length=255), nullable=False),
            # ... more columns ...
        )


    # contraction operations
    # .../alembic_migrations/versions/liberty/contract/4405aedc050e_hierarchical_binding.py

    def upgrade():

        op.drop_constraint(fk_name_dvr[0], 'ml2_dvr_port_bindings', 'foreignkey')
        op.drop_column('ml2_dvr_port_bindings', 'cap_port_filter')
        op.drop_column('ml2_dvr_port_bindings', 'segment')
        op.drop_column('ml2_dvr_port_bindings', 'driver')

        # ... more DROP instructions ...

The two scripts would be present in different subdirectories and also
part of entirely separate versioning streams, discussed in the section
below "New Migration Layout".  The "expand" operations are in the
"expand" script, and the "contract" operations are in the "contract"
script.

The data migrations are removed, as these are expected to
generally not occur within schema migrations any more.  However,
the approach remains compatible with allowing "safe" data migrations to be
manually placed within the expand or contract scripts if deemed appropriate
in some cases.

For the time being, until live data migration is accepted in Neutron, data
migration rules belong to one of script subtrees.

New Migration Layout
--------------------

With Alembic's new capabilities, we can propose a new structure for
Neutron's migration files that is compatible with "expand" / "contract"
while at the same time remains compatible with the existing stream
of Alembic migration files in Neutron.   A new directory/branch
structure will be laid out which allows all versions/streams to be apparent::


    neutron/db/migration/alembic_migrations/...

    ...versions/
                 <existing version>.py
                 <existing version>.py
                 <existing version>.py
                 ...

    versions/liberty/
    versions/liberty/expand/
                                    <expansion script>.py
                                    <expansion script>.py
                                    ...

    versions/liberty/contract/
                                    <contract script>.py
                                    <contract script>.py
                                    ...

    versions/M_release/
    versions/M_release/expand/
    versions/M_release/contract/
    ... etc

Above, the existing /versions/ directory with all of its current migration
scripts remains intact; these versions are still the scripts that take a
Neutron database up through Kilo at least.  Following those, a new
series of subdirectories are added, organized among major Openstack releases,
and within each subdirectory, the "expand" and "contract" series of scripts
are themselves separate.

The series of scripts within ``/expand/`` and ``/contract/`` are
themselves originating from independent "roots"; that is, the "down"
revision for the bottommost script in each directory is ``None``.

The production of these scripts is supported by the ``alembic revision`` command,
which now includes options to place files in specific directories as well
as what the "down revision" of a given revision is to be, including
that it may be a "root", thus allowing the creation of new branches
and roots.

Cross-Branch Dependencies
--------------------------

To accommodate for the fact that the scripts in ``liberty/expand``
can't be run until all the old scripts in ``versions/`` have run, as
well as that individual scripts in ``liberty/contract/`` can't run until
their correpsonding "expand" has run, Alembic's cross-branch dependency
feature will be used.   From a DAG point of view, this is the same as a script
declaring another one as a dependency, but from Alembic's perspective the
script is not considered to be any kind of "down revision"; only a script
whose version must be invoked within the target schema before the current
one can be run.   They are indicated in Alembic scripts as a separate directive::

    # revision identifiers, used by Alembic.
    revision = '2a95102259be'
    down_revision = '29f859a13ea'
    branch_labels = None
    depends_on=('55af2cb1c267', '4fcb78af5a01')

By establishing "depends_on", a particular script indicates what migration
scripts in other branches need to be run first, before this one can.
When we instruct Alembic to invoke this migration, it will ensure that
all dependency scripts are run first.   It is expected that
the automated script creation tool will be able to build out these
directives automatically.

Alembic branch dependencies are discussed in the Alembic documentation
referred to in the References section.

Branch Labels
-------------

Alembic also now provides for "branch labels", meaning that in addition
to having our migration files in different directories, versioned across
independent branches with independent roots, we can also apply one or
more "labels" to a branch as a whole which is then addressable using Alembic's
command line tools.
Whereas in Neutron today we can see some migration scripts that are intentionally
named ``juno_release.py`` and ``kilo_release.py``, we can apply these names
to the branches as a whole.    Like branch dependencies, these are also
indicated as directives within migration scripts; however, the branch
label only need be present in *any* single revision script within the
branch.  Typically, the first script within the branch is a good
choice for placing a label::

    # revision identifiers, used by Alembic.
    revision = '2a95102259be'
    down_revision = None  # because we are a "root"
    branch_labels = ('liberty_expand', 'release_expand')
    depends_on='55af2cb1c267'

So above, we would apply names such as ``"liberty_expand"`` and
``"liberty_contract"`` to ``liberty/expand`` and ``liberty/contract`` branches,
appropriately.  This allows Alembic commands to be run which refer to the
branch as a whole, such as::

    alembic upgrade liberty_expand@head

Where above, all migrations up until the ``liberty_expand`` branch will
be run (including dependent versions from the old series of migration
files first, if not already run).   This will allow Neutron's command suite
to accommodate specific target points within the new versioning scheme
without the need to become aware of specific revisions.

If labels that are agnostic of "release" are desired, such as a branch
that indicates "run all the expand steps up to the current release", we
can add additional "latest release" labels that move to new branches
as new releases are established.


New Neutron DB Commands
-----------------------

Right now, Neutron allows database upgrades running the ``neutron-db-manage``
script, which links into Alembic's own ``upgrade`` command.   This script
will be enhanced to allow for running individual migration streams by taking
advantage of new argument forms that are part of Alembic.  The ``alembic upgrade``
command will still be used but will now be passed the appropriate branch
labels specific to the target operation, such as ``neutron-db-manage expand``
or ``neutron-db-manage contract``.

Automation of Scripting
-----------------------

The previous sections essentially make possible the entire "expand" / "contract"
workflow completely, in such a way that workflow from Neutron's existing
Alembic versioning scripts is maintained without any backwards incompatibility.
However, the addition of new migration scripts would at first be available
only by manually targeting each portion of the workflow individually.

We instead can enhance Neutron's use of Alembic "autogenerate" such
that a single revision autogenerate step can produce multiple files as
needed; a migration that includes both "expand" and "contract"
directives would generate two separate scripts.

Right now, Nova OSM makes use of Alembic autogenerate in order to derive
information about how a target database differs from the model established
in code.  It uses a public method ``compare_metadata()`` to achieve this;
``compare_metadata()`` returns a simple list of "diffs" which refer to changes
in schema objects like tables, columns, and constraints.  Nova OSM then
keys "operational" objects such as ``AddTable``, ``DropColumn``, ``AddConstraint``,
etc.  These "operational" objects then link back into Alembic's API,
associated with corresponding "operation" constructs in Alembic such
as ``op.create_table()``, ``op.drop_column()``, ``op.add_constraint()``.

Nova's OSM is basically consuming an Alembic "autogenerate diff" stream
and streaming it into an Alembic "run operations" stream.   It follows that
Alembic can provide infrastructure such that an autogenerate
diff stream can be supplied directly as a migration operation stream.
Both the "live" OSM approach of Nova and the "scripted" approach
proposed here can consume this same operation stream, partition it
based on operation type into "expand" and "contract" streams, and then
direct those streams either to a live database context for "live" migrations
*or* to a series of migration scripts for scripted migrations.

The ``alembic revision`` command will also be opened up such that a plugin
may establish an open-ended series of revision scripts generated from
portions of these operational streams.   The end result will be that
a single call to ``alembic revision --autogenerate`` as performed by
Neutron developers today will generate separate "expand" and "contract" scripts
directly.

These new APIs are already  underway in upstream development branches and are
tracked by separate Alembic issues (see References).  By closely linking the
implementations for "live" migrations and "scripted"
migrations, it is hoped that the majority of ongoing effort within OSM
can contribute to both approaches simultaneously, thus reducing the risk
that work is wasted either if one or the other approach is abandoned
or that improvements and workflows begin to diverge if both approaches
remain in active use.


Data Model Impact
-----------------

Expand/contract workflow itself has no direct impact on the data model.   The
other aspects of online schema migrations, namely support of multiple
versions of a schema simultanouesly as well as moving data between those
structures at runtime have an enormous impact; however, that's outside the
scope of this document.

REST API Impact
---------------

none


Security Impact
---------------

none


Other End User Impact
---------------------

End users will be using a modified workflow when schema upgrades
are performed, running "expand" and "contract" steps separately and
at the appropriate time.

In terms of backup procedures, there is no difference. Database always
represents some specific subset of head revisions (the only difference
between the proposed feature and the current state is that the subset
has more than one element).

If/once we adopt live data migration in Neutron, it won't change a lot
in terms of database backups either way. The only significant thing is
that the same logical object could now be represented by multiple
versions of database rows, depending on whether migration for the
object is complete. Anyway, backups would still work as usual.


Performance Impact
------------------

none

Developer Impact
----------------

Developers should continue to use ``alembic revision --autogenerate``
in order to create new migration scripts.  This operation will create
multiple scripts, so to the degree that developers need to manually
tune these scripts, they'll be dealing with more than one script.  Since
data migrations will generally no longer be within these scripts, and
also since we now have the ability to render custom directives via
autogenerate, it is hoped that pretty much anything Nova's "live" OSM
approach can automate can also be completely automatic within the
"scripted" approach as well.


Alternatives
------------

Live migrations were originally proposed as a replacement for
SQLAlchemy-Migrate, which has the additional issues of a very rigid
and unworkable numbering scheme as well as verbose migration scripts
that rely heavily on full table declarations and reflection.  These
are not issues for projects that already use Alembic, as Alembic was
designed to solve these problems among others.

Live migration also offers the advantages that no script
at all needs to be generated or committed into the source repository,
and also that because there is literally no way to alter how
migrations will proceed on a per-change basis, developers are not
given any opportunity to inadvertently produce a migration that is
non-expansive or to inappropriately write statements in a migration
that result in performing a data migration.  This is noted
as allowing a "purely declarative" approach to migrations, where the
model code is all that's needed to indicate how to get to the new
schema.

However, this advantage only works under limited circumstances. While
simple cases are easily automated by both approaches, the issue of
accommodating for special cases, unsupported features, and variability
in support and/or reliability on various backends is not addressed
by the "live" approach. Live migrations only offers in such cases that
the upstream migration system must be modified to accommodate for the
target case, or the application must be modified to no longer require
such a schema migration. Special cases include changes to complex
types like ENUMs and precision numerics, operations involving CHECK
constraints and some kinds of server defaults, special constructs such
as indexes that use vendor-specific extensions, and even simple
things like changes of table or column name that can't be
distinguished from an add/drop of two separate objects.

The availability and reliability of the reflection and autogenerate
features on backends is not necessarily consistent, nor can the
Alembic / SQLAlchemy projects make any such guarantee. In particular,
less common backends such as that of IBM DB2, which is published
independently of SQLAlchemy or Alembic, may not support some
operations correctly at all, and it is not currently known to what
extent autogenerate and reflection produce accurate results,
especially in thorny cases such as indexes, unique constraints, and
column types.  Alembic's autogenerate feature was not intended to be
used in the way that live migrations does, and it's a risky assumption
that it will produce correct results perfectly under all cases on
thousands of production systems.

While the "scripted" OSM approach maintains reliance upon explicit migration
scripts that must be checked in and occasionally edited, the advantage to this
approach is that the sequence of migration steps to be run are produced
just once, up front, in a controlled environment.  Unusual migrations against
special types or other constructs are again a non-issue as they can be
scripted explicitly as needed.  These steps can then be
carefully reviewed and tested by developers and then shipped, where there
is no risk of them doing something entirely different when run against
a backend of a lesser-used vendor or with unusual configurations.  They
also maintain the advantage that operator-maintained schema
structures are unaffected; the "live" approach documents that operators
would need to re-create their own schema structures after a "contract" is run.

In order to maintain that scripted migrations stay appropriately
expansive / contractual and without inappropriate data migrations in
the face of developer intervention, we should require that developers
use the autogenerated migration scripts that are generated for them as
is, and that they don't modify these scripts except to support
operations that aren't supported as automatable migrations.  Schema
migrations should be tested as part of the CI process including that
the "online upgrades" are tested against previous API versions, and
migration scripts of course go through the usual Gerrit code review
process; identifying migrations that are non-expansive or are data
migrations is not difficult.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Mike Bayer

Other contributors:
  Ann Kamyshnikova
  Henry Gessau
  Ihar Hrachyshka

Work Items
----------

* update neutron-db-manage to support upgrade for multiple heads.
* update neutron-db-manage revision to generate multiple scripts.
* get alembic updated to include update stream generated.
* use the alembic autogenerate feature to filter operations into subtrees.
* implement new expand/contract neutron-db-manage commands.
* document the change in the upgrade flow in user and developer docs.
* introduce testing for expand-only schema upgrade.

Dependencies
============

Upstream changes to Alembic for the autogenerate integration aspect.

Testing
=======

Functional tests should include that "expand" migrations are run and that
the previous version of the API still works fully against an expanded
migration.


Documentation Impact
====================


User Documentation
------------------

Expand/contract workflow will need to be documented.


Developer Documentation
-----------------------

Usage of autogenerate along with expand/contract workflow can be documented.

References
==========

.. [#] Alembic's Branching Model
 http://alembic.readthedocs.org/en/latest/branches.html

.. [#] Online Schema Migrations in Nova
 http://specs.openstack.org/openstack/nova-specs/specs/liberty/approved/online-schema-changes.html


.. [#] Nova's Overall Online Upgrade approach
 http://docs.openstack.org/developer/nova/devref/upgrade.html

.. [#] Operations as Objects
 https://bitbucket.org/zzzeek/alembic/issue/302/operations-as-objects

.. [#] Extensible Revision / Autogenerate strategies
 https://bitbucket.org/zzzeek/alembic/issue/301/extensible-revision-autogenerate

.. [#] Neutron patch to rearrange migration directory into subtrees
 https://review.openstack.org/194198
