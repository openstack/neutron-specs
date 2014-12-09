..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Proposal for Neutron Services spin-off(s)
=========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/services-split

This proposal deals with the technical aspects of splitting the neutron repo
into four repos, one for basic L2/L3 plumbing, and one each for LBaaS, FWaaS,
and VPNaaS.  The repos will be referred to as "neutron", "vpnaas", "fwaas",
and "lbaas" within this spec, until project names for the repos are selected.

This spec does not cover governance concerns.  For that issue, refer to the
mailing list thread at the end of this document, or talk to the Networking
Program PTL.

Problem Description
===================

The following blurb is shamelessly stolen from an email thread started by
Mark McClain:

Over the last several months, the members of the Networking Program have been
discussing ways to improve the management of our program.  When the Quantum
project was initially launched, we envisioned a combined service that included
all things network related.  This vision served us well in the early days as
the team mostly focused on building out layers 2 and 3; however, we have run into
growth challenges as the project started building out layers 4 through 7.
Initially, we thought that development would float across all layers of the
networking stack, but the reality is that the development concentrates around
either layer 2 and 3 or layers 4 through 7.  In the last few cycles, we have also
discovered that these concentrations have different velocities and a single
core team forces one to match the other to the detriment of the one forced to
slow down.

Proposed Change
===============

Going forward we want to divide the Neutron repository into multiple separate
repositories lead by a common Networking PTL. The current mission of the
program will remain unchanged.

* The repo will be split via infra scripts which will preserve history for all
  changes.

* The existing lbaas code will move into the lbaas repo, to be supported
  by that team going forward.

* The existing fwaas code will move into the fwaas repo, to be supported
  by that team going forward.

* The existing vpnaas code will move into the vpnaas repo, to be supported
  by that team going forward.

* Unlike core neutron, vendor code will remain in each of the service repos,
  while we are still attempting to build community/critical mass.

* The service repos will include Neutron as a requirement.  This may flip
  for the very short-term until the L3 agent is refactored to not derive
  from the firewall class.  Package maintainers *must* install the services
  along with Neutron for any upgrade from Icehouse/Juno to Kilo.

* The service repos will support the same tox environments as neutron.

* Bugfixes for services will need to continue to be backported into the Neutron
  stable branches for several releases.  Backporting should preserve change-id
  and commit message.

* In Kilo, the API client/CLI will be via the neutronclient.  This does not
  preclude creating a standalone client/API, but it will not be a split
  requirement.

* This is about splitting repos.  Nothing in this spec should be viewed as
  deprecating any currently shipping service code, even if it moves repos.

* Third-party CIs must point at the relevant repo.

* In Kilo, the four repos will play nicely together on master, but not
  between any other versions/branches.

Kilo-1 (to occur during neutron meetup, 12/8)
---------------------------------------------

* Before the split, an infra/project-config change for the new repos, and
  new core teams will be created.

* After the split, all WildcardaaS code and db models will be in their new repo,
  with history preserved.

* After the split, no lbaas/fwaas/vpnaas services code to be in the neutron
  repo, with non-relavant history pruned (exception: extensions and l3 agent,
  see below.)

* Extensions will stay in neutron until post rest-refactor, to avoid breaking
  the co-gate.

* Service repos will continue to use the database defined in neutron.conf,
  but will have their own tables and alembic chains.  Foreign keys between
  migration chains will not be allowed; existing foreign key relationships
  should be dealt with during kilo refactors/cleanup.

* Service repos will use their own config file, located under /etc/neutron.

* Code merged onto the existing 'feature/lbaasv2' neutron feature branch will
  be merged into the lbaas repo as part of the split.

* All outstanding gerrit reviews for services, currently submitted against
  Neutron, will have to be abandoned and resubmitted against the relevant
  service repo. Spec reviews will remain as-is, as the spec repo is not changing.

* Tox will pass cleanly in all projects immediately post-split, except for the
  silly never-passing py3 environments.

Kilo-2
------

* Deadline for third-party CIs to be updated for new repos.

* Post REST-refactor, extensions will be moved to the service repos.

* Refactor L3 agent to not depends on services code.  This work is part of
  the current Kilo L3 refactor.

Kilo-3
------

* No more foreign keys crossing alembic chains.

* L3 agent refactor complete, break circular exception between neutron/services
  for the l3_agent.

* API tests into each service repo.

L release
---------

* Anywhere that services import neutron, evaluate whether it is using neutron
  as a library appropriately, or if it implies a missing interface in the
  Neutron API.

M release
---------

* Services have their own REST endpoints, databases, config files, and CLI/API
  clients.

Data Model Impact
-----------------

Services data models will be moved to the service repos, and removed from
neutron.

The service repos will each have their own database migration chains.

An initial db migration state will be created by starting from the neutron
db state as of the split and stripping non-service related tables.

Foreign keys referencing neutron models will be continue to store the uuid of the
neutron object, but the associated orm reference will be converted to query the
neutron database.  Existing foreign key associations: Firewall, none.
LBaaS: VIP model references Port.  VPN: VPNService references subnet and router.

REST API Impact
---------------

The REST API will be identical immediately before and after the split.

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

None.

IPv6 Impact
-----------

None.

Other Deployer Impact
---------------------

The new service projects will have their own database tables, migration chains,
and config files.  In addition, by the end of Kilo, neutron will need to load
all of the services API extensions from the out-of-neutron repos.

For Kilo, neutron will assume that the services repos exist, and include the
path to their API extensions in the neutron.conf file by default
(api_extensions_path).

In the upgrade scenario, the REST controller will bounce, but active services
(load balancers, etc) will remain active.


Developer Impact
----------------

Anyone importing neutron.services will have to import the new project modules
instead.

Patches might need to be resubmitted against the correct repo.

Community Impact
----------------

This will enable teams focused exclusively on one or more advanced services to
make a bigger impact and ensure progress.

Alternatives
------------

* Do nothing and keep it all in one repo.  This is the status quo, and is
  untenable.

* Neutron split into two repos, one neutron, one advanced services.  The benefits
  of this approach are a larger initial community, a simpler split, and
  somewhere for new advanced services to "incubate" in-tree other than Neutron.
  The downsides are similar to continuing to have services in
  Neutron itself: less focus, larger chance of the priorities of a popular
  service overriding a less popular or newer one, and less separation of concerns.

* Services to stackforge.  Completely separate governance, must be incubated.

* Services split with its own REST server endpoint.  More separation of concerns,
  more work required.

* Services have their own databases entirely.  Enforces better separation, but
  likely overkill before there are separate REST endpoints.

* Modify gerrit to allow different core teams in one repo.  This does not
  encourage separation of concerns, and gerrit does not support this today.

* Split repos but continue to use neutron db (own tables, own chains)

* The service repos will need to use neutron as a library, and should have a
  dependency on Neutron. But, to accomodate seamless upgrades from
  Icehouse/Juno to Kilo, the dependencies could be flipped for one cycle:

  * For Kilo, neutron will have a dependency on each of the service repos, to
    make install of the services automatic with Neutron for Kilo.  Also, only
    for Kilo, neutron will have a hacking rule to prevent importing any of the
    service repos, in preparation.

  * For Kilo, to prevent a circular dependency, the services can assume that
    the neutron code is installed, and import it as needed.

  * For L, the dependency will be removed from neutron, and each of the service
    repos will add a neutron dependency of their own.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~dougwig

LBaaS assignee:
  https://launchpad.net/~dougwig

FWaaS assignee:
  https://launchpad.net/~snaiksat

VPNaaS assignee:
  https://launchpad.net/~pcm

Other contributors:
  https://launchpad.net/~mestery

Work Items
----------

Work items for the split:

* Identify files for each repo.

* Adapt oslo graduation script for git split.

* Merge in lbaasv2 feature branch.

* Adjust imports in new repos.

* Add requirements to each project.

* Add hacking rule to neutron to prevent service import, with the exception
  of the existing import in the L3 agent.

* Verify or add neutron's ability to load out-of-tree service plugins.

* Create initial services db migration files, make sure all get applied.

* Fix references to neutron in various files (e.g. README)

* Finalize project names

* Infra patch to create new repos/groups

* Get unit tests passing cleanly.

Work items that are implied in doing the split, but which will happen separately/afterwards:

* Anywhere that services import neutron, evaluate whether it is using neutron
  as a library appropriately, or if it implies a missing interface in the
  Neutron API.

* Refactor L3 agent to not reach into the guts of services.

* API tests into each service repo.


Dependencies
============

* Infra creating separate repos.

* REST refactor not colliding at the same time.  This needs to happen before
  or after.


Testing
=======

* Unit tests will split between repos, matching the code split.

* Tempest tests will initially remain unchanged, as the service endpoint will
  be identical before and after the split.  Setup steps that touch db and/or
  config files may need to be updated to reflect new locations.

* Advanced services test will be removed from the "integrated gate". load
  balancing & friends will co-gate with neutron only, and not anymore with
  nova, cinder and the others.

Tempest Tests
-------------

Unchanged, unless tests are in neutron by the split, then they will move.

Functional Tests
----------------

Tests which load extensions by their extension namespace will be updated for
the new paths.

API Tests
---------

Unchanged, unless tests are in neutron by the split, then they will move.


Documentation Impact
====================

User Documentation
------------------

Documentation referencing neutron.conf and the neutron db will need to be
modified to reflect the new config file and database.

Developer Documentation
-----------------------

Documentation referencing neutron.conf and the neutron db will need to be
modified to reflect the new config file and database.


References
==========

* http://lists.openstack.org/pipermail/openstack-dev/2014-November/050961.html

* Summit etherpad - https://etherpad.openstack.org/p/neutron-services

* Repo creation - https://review.openstack.org/#/c/138475/

* Governance change - https://review.openstack.org/138479
