..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Networking-l2gw Scorecard
=========================

Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or relies on proprietary backends?

  Networking-l2gw implements its own set of Neutron API extensions on top of
  the Neutron core framework and it does so by using the service plugin model.
  The API exposed has open source implementations, and it provides a pluggable
  mechanism for proprietary backends.

.. _N1:

* N1. Does the project integrate/use neutron-lib?

  Yes. The migration report shows that there are currently ~450 total imports.
  Neutron is imported ~100 times and Neutron-lib only ~10 times, for a migration
  percentage of 11.2000%. No periodic job against neutron-lib seems available.

.. _N2:

* N2. Do project members actively contribute to help neutron-lib achieve its
  goal?

  None of the project core members have merged anything meaningful into neutron-lib
  (source: https://review.openstack.org/#/q/project:openstack/neutron-lib+status:merged,75).

.. _N3:

* N3. Do project members collaborate with the core team to enable subprojects
  to loosely integrate with the Neutron core platform by helping with the definition
  of modular interfaces?

  Not particularly.

.. _N4:

* N4. How does the project provide networking services? Does it use modular interfaces
  as provided by the core platform?

  Yes.

.. _N5:

* N5. If the project provides new API extensions, have API extensions been discussed
  and accepted by the Neutron drivers team? Please provide links to API specs, if
  required.

  Not exactly. The project was one of the initial projects created during the early
  days of the Big Tent and Neutron decomposition. The API was accepted/approved by
  a smaller group of people who wanted to make progress on a topic that was largely
  stalled due to lack of community consensus.

Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (links to logs.openstack.org).

  Yes.

  * https://github.com/openstack/networking-l2gw/blob/master/tox.ini

.. _D2:

* D2. If the project provide API extensions, does the project have an
  api-ref tox target, functional and continously working? Provide proof
  (links to logs.openstack.org).

  There is no API reference published, though the API is detailed in
  its spec document.

  * https://raw.githubusercontent.com/openstack/networking-l2gw/master/specs/kilo/l2-gateway-api.rst

.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continously working? Provide proof.

  No.

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.

  There is not a lot of documentation available.

  * http://docs.openstack.org/developer/networking-l2gw/


Continuous Integration
----------------------

.. _C1:

* C1. Does the project have a Grafana dashboard showing historical trends of
  all the jobs available? Provide proof (links to grafana.openstack.org).

  No.

.. _C2:

* C2. Does the project have CI for unit coverage? Provide proof (links to
  logs.openstack.org)

  Yes.

  * http://status.openstack.org/openstack-health/#/g/project/openstack~2Fnetworking-l2gw

.. _C3:

* C3. Does the project have CI for functional coverage? If so, does it include
  DB migration and sync validation?

  No. but DB migration and sync validation is provided by its unit job.

  * http://logs.openstack.org/99/380899/1/gate/gate-networking-l2gw-python35-db/f3af348/testr_results.html.gz

.. _C4:

* C4. Does the project have CI for fullstack coverage?

  No.

.. _C5:

* C5. Does the project have CI for Tempest coverage? If so, specify nature
  (API and/or Scenario).

  Latest Bot proposal shows only unit validation. Even though the project has
  API and scenario tests they are exercised with downstream CI because of lack
  of an pure software implementation of L2GW services.

  * https://review.openstack.org/#/c/357703/

.. _C6:

* C6. How does a project validate upgrades on a continuous basis? Does
  the project require or support CI for Grenade coverage?

  Yes. But it has none.

.. _C7:

* C7. Does the project provide multinode CI?

  No.

.. _C8:

* C8. Does the project support Python 3.x? Provide proof.

  Yes.

  http://logs.openstack.org/03/357703/7/gate/gate-networking-l2gw-python35-db/b834ee7/testr_results.html.gz


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

  * https://github.com/openstack/networking-l2gw/blob/master/tox.ini#L10

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?

  Yes.

  * https://github.com/openstack/requirements/commit/574736a84b9218e1d7ea860f82cb248975b7c1ca


Stable backports
----------------

.. _S1:

* S1. Does the project have stable branches and/or tags? Provide history of
  backports.

  Yes, stable maintainance is responsibility of the neutron-stable team.


Client library
--------------

.. _L1:

* L1. If the project requires a client library, how does it implement CLI and
  API bindings?

  There are Neutron CLI extensions but they have not been ported over to OSC.

  * https://github.com/openstack/networking-l2gw/tree/master/networking_l2gw/l2gatewayclient


Scorecard
---------

+---------------+
| Scorecard     |
+===============+
| N0_ |    Y    |
+---------------+
| N1_ |    Y    |
+---------------+
| N2_ |    N    |
+---------------+
| N3_ |    N    |
+---------------+
| N4_ |    Y    |
+---------------+
| N5_ |    N    |
+---------------+
| D1_ |    Y    |
+---------------+
| D2_ |    N    |
+---------------+
| D3_ |    N    |
+---------------+
| D4_ |    N    |
+---------------+
| C1_ |    N    |
+---------------+
| C2_ |    Y    |
+---------------+
| C3_ |    N    |
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

Closing the gap on all the remaining unmet criteria in time for Ocata-1
(Nov 14 2016) seems challenging. Progress lacked during the Newton cycle.
It is probably time for the core team to be rebooted.
