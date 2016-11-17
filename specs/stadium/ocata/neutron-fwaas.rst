..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Neutron-fwaas Scorecard
=======================

Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or rely on proprietary backends?

  Neutron-fwaas implements its own set of Neutron API extensions on top of
  the Neutron core framework and it does so by using the service plugin model.
  The API exposed has open source implementations, and it provides a pluggable
  mechanism for proprietary backends.

.. _N1:

* N1. Does the project integrate/use neutron-lib?

  Yes. The migration report shows that there are currently ~400 total imports.
  Neutron is imported ~100 times and Neutron-lib only ~20 times, for a migration
  percentage of 18.045%. The project has periodic validation with neutron-lib.

  * https://etherpad.openstack.org/p/neutron_lib_fwaas_punchlist

.. _N2:

* N2. Do project members actively contribute to help neutron-lib achieve its
  goal?

  Yes.  For example: https://review.openstack.org/389825

.. _N3:

* N3. Do project members collaborate with the core team to enable subprojects
  to loosely integrate with the Neutron core platform by helping with the definition
  of modular interfaces?

  The team has worked successfully in delivering the `L3 agent extension framework <https://blueprints.launchpad.net/neutron/+spec/l3-agent-extensions>`_, that
  enabled the team to break the fork of the L3 agent. However, this framework
  should be contributed to neutron-lib to help increase the positive score of
  N2, which is in progress.  See:
  `Migrate neutron agent extensions to neutron-lib <https://review.openstack.org/#/c/385045/_>`

.. _N4:

* N4. How does the project provide networking services? Does it use modular interfaces
  as provided by the core platform?

  Yes.

.. _N5:

* N5. If the project provides new API extensions, have API extensions been discussed
  and accepted by the Neutron drivers team? Please provide links to API specs, if
  required.

  The fwaas v1 and v2 API have been widely discussed and accepted.


Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (links to logs.openstack.org).

  Yes.

  * https://github.com/openstack/neutron-fwaas/blob/master/tox.ini

.. _D2:

* D2. If the project provide API extensions, does the project have an
  api-ref tox target, functional and continously working? Provide proof
  (links to logs.openstack.org).

  The API reference for the v2 API has not merged at the time of writing, see
  `FWaaS v2 API reference <https://review.openstack.org/#/c/391338/>`_

  * http://developer.openstack.org/api-ref/networking/

.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continously working? Provide proof.

  Yes.

  * https://github.com/openstack/neutron-fwaas/blob/master/tox.ini
  * http://docs.openstack.org/releasenotes/neutron-fwaas/

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.

  Developer documentation is sparse, user documentation is stale.

  * http://docs.openstack.org/developer/neutron-fwaas/


Continuous Integration
----------------------

.. _C1:

* C1. Does the project have a Grafana dashboard showing historical trends of
  all the jobs available? Provide proof (links to grafana.openstack.org).

  Yes.

  * http://grafana.openstack.org/dashboard/db/neutron-fwaas-failure-rates

.. _C2:

* C2. Does the project have CI for unit coverage? Provide proof (links to
  logs.openstack.org)

  Yes.

  * http://status.openstack.org/openstack-health/#/g/project/openstack~2Fneutron-fwaas

.. _C3:

* C3. Does the project have CI for functional coverage? If so, does it include
  DB migration and sync validation?

  Yes.

  * http://status.openstack.org/openstack-health/#/job/gate-neutron-fwaas-dsvm-functional

.. _C4:

* C4. Does the project have CI for fullstack coverage?

  No.

.. _C5:

* C5. Does the project have CI for Tempest coverage? If so, specify nature
  (API and/or Scenario).

  Latest Bot proposal shows a `grim picture <https://review.openstack.org/#/c/378852/>`_.
  No API/Scenario coverage for the v2 API, though v1 has it.

  * http://logs.openstack.org/52/378852/1/check/gate-neutron-fwaas-dsvm-tempest/68423cd/testr_results.html.gz
  * Coverage for v2 API tests: https://review.openstack.org/#/c/391320/ (WIP)
  * Coverage for v2 scenario tests: https://review.openstack.org/#/c/391392/ (WIP)

.. _C6:

* C6. How does a project validate upgrades on a continuous basis? Does
  the project require or support CI for Grenade coverage?

  No, but it does in the experimental queue.  Some tests need to be identified as smoke tests.

.. _C7:

* C7. Does the project provide multinode CI?

  No, but it is in progress.

  * https://review.openstack.org/#/c/386933/

.. _C8:

* C8. Does the project support Python 3.x? Provide proof.

  Yes.

  * http://logs.openstack.org/52/378852/1/check/gate-neutron-fwaas-python35/b321bc5/testr_results.html.gz


Release footprint
-----------------

.. _R1:

* R1. Does the project adopt semver?

  Yes.

.. _R2:

* R2. Does the project have release deliverables? Provide proof as available
  in the `release repo <http://git.openstack.org/cgit/openstack/releases/tree/>`_.

  Yes, the release is responsibility of the neutron-release team.

.. _R3:

* R3. Does the project use upper-constraints?

  Yes.

  * https://github.com/openstack/neutron-fwaas/blob/master/tox.ini#L11

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?

  Yes.

  * https://github.com/openstack/requirements/commit/1d545edbebfff2e8983d6cab24a92c32636dd6bf


Stable backports
----------------

.. _S1:

* S1. Does the project have stable branches and/or tags? Provide history of
  backports.

  Yes, stable maintenance is responsibility of the neutron-stable-maint team.


Client library
--------------

.. _L1:

* L1. If the project requires a client library, how does it implement CLI and
  API bindings?

  There are Neutron CLI and API bindings for v1, none released for v2 yet.


Scorecard
---------

+---------------+
| Scorecard     |
+===============+
| N0_ |    Y    |
+---------------+
| N1_ |    Y    |
+---------------+
| N2_ |    Y    |
+---------------+
| N3_ |    Y    |
+---------------+
| N4_ |    Y    |
+---------------+
| N5_ |    Y    |
+---------------+
| D1_ |    Y    |
+---------------+
| D2_ |    N    |
+---------------+
| D3_ |    Y    |
+---------------+
| D4_ |    N    |
+---------------+
| C1_ |    Y    |
+---------------+
| C2_ |    Y    |
+---------------+
| C3_ |    Y    |
+---------------+
| C4_ |    N    |
+---------------+
| C5_ |    N    |
+---------------+
| C6_ |    N    |
+---------------+
| C7_ |    N    |
+---------------+
| C8_ |    Y    |
+---------------+
| R1_ |    Y    |
+---------------+
| R2_ |    Y    |
+---------------+
| R3_ |    Y    |
+---------------+
| R4_ |    Y    |
+---------------+
| S1_ |    Y    |
+-----+---------+
| L1_ |    N    |
+-----+---------+


Final remarks
-------------

At the time of writing the project scores positively in 17 out of 22
criteria. Even though the fwaas team has made quite a progress during the
Newton cycle, closing the gap on all the remaining unmet criteria in time
for Ocata-1 (Nov 14 2016) seems challenging.
