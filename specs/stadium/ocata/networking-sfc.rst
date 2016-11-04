..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Networking-sfc Scorecard
========================

Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or relies on proprietary backends?

  Networking-sfc implements its own set of Neutron API extensions on top of
  the Neutron core framework and it does so by using the service plugin model,
  The API exposed has open source implementations, and it provides a pluggable
  driver mechanism for multiple backends.

.. _N1:

* N1. Does the project integrate/use neutron-lib?

  Yes. It only imports ~10% of the neutron-related imports required. There is no
  periodic job to test against neutron-lib master changes.

.. _N2:

* N2. Do project members actively contribute to help neutron-lib achieve its
  goal?

  No, there is no tangible evidence.

.. _N3:

* N3. Do project members collaborate with the core team to enable subprojects
  to loosely integrate with the Neutron core platform by helping with the definition
  of modular interfaces?

  Team members helped review some of the L2 extensions related specs and patches.

.. _N4:

* N4. How does the project provide networking services? Does it use modular interfaces
  as provided by the core platform?

  Yes, only recently the team managed to kill the OVS agent fork they were relying on.

  * https://review.openstack.org/#/c/351789/

.. _N5:

* N5. If the project provides new API extensions, have API extensions been discussed
  and accepted by the Neutron drivers team? Please provide links to API specs, if
  required.

  The project currently exposes two APIs: flow classification and Port chaining both
  via two separate service plugins. Minimal oversight on the latter was provided by
  the Neutron team at the project inception.


Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (links to logs.openstack.org).

  Yes.

  * http://docs.openstack.org/developer/networking-sfc/

.. _D2:

* D2. If the project provide API extensions, does the project have an
  api-ref tox target, functional and continously working? Provide proof
  (links to logs.openstack.org).

  Yes.

  * https://github.com/openstack/networking-sfc/blob/master/tox.ini#L136

.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continously working? Provide proof.

  No.

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.

  There is some developer documentation (in the form of design documents),
  deployment documentation, and some user documentation in the form of a
  list of client side extensions. Some material is available on the
  wiki.o.o, which ideally would be migrated over and put under version control.

  * http://docs.openstack.org/developer/networking-sfc/
  * http://docs.openstack.org/newton/networking-guide/config-sfc.html

Continuous Integration
----------------------

.. _C1:

* C1. Does the project have a Grafana dashboard showing historical trends of
  all the jobs available? Provide proof (links to grafana.openstack.org).

  No.

  * https://review.openstack.org/#/c/386081/

.. _C2:

* C2. Does the project have CI for unit coverage? Provide proof (links to
  logs.openstack.org)

  Yes.

  * http://status.openstack.org/openstack-health/#/g/project/openstack~2Fnetworking-sfc

.. _C3:

* C3. Does the project have CI for functional coverage? If so, does it include
  DB migration and sync validation?

  Yes. DB migration and sync validation introduced recently.

  * https://review.openstack.org/#/c/354359/
  * http://logs.openstack.org/59/354359/3/check/gate-networking-sfc-python35-db/cf7dd20/testr_results.html.gz

.. _C4:

* C4. Does the project have CI for fullstack coverage?

  No.

.. _C5:

* C5. Does the project have CI for Tempest coverage? If so, specify nature
  (API and/or Scenario).

  API and Scenario. Non voting.

.. _C6:

* C6. How does a project validates upgrades on a continuous basis? Does
  the project require or support CI for Grenade coverage?

  No upgrade testing on a continuous basis.

.. _C7:

* C7. Does the project provide multinode CI?

  No.

.. _C8:

* C8. Does the project support Python 3.x? Provide proof.

  Yes.

  * http://logs.openstack.org/38/376538/3/check/gate-networking-sfc-python35-db/c05fa62/


Release footprint
-----------------

.. _R1:

* R1. Does the project adopt semver?

  Yes.

.. _R2:

* R2. Does the project have release deliverables? Provide proof as available
  in the `release repo <http://git.openstack.org/cgit/openstack/releases/tree/>`_.

  Yes.

  * http://git.openstack.org/cgit/openstack/releases/tree/deliverables/_independent/networking-sfc.yaml

.. _R3:

* R3. Does the project use upper-constraints?

  Yes.

  * https://github.com/openstack/networking-sfc/blob/master/tox.ini#L10

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?

  Yes.

  * https://github.com/openstack/requirements/commit/2afcaea9a8c0363173f215c2316b59985a981d0e


Stable backports
----------------

.. _S1:

* S1. Does the project have stable branches and/or tags? Provide history of
  backports.

  Liberty and Mitaka available. Backports are under the control of the neutron
  stable team. The subproject follows some release cadence that is not in sync
  with neutron, and this must be rectified, ASAP.


Client library
--------------

.. _L1:

* L1. If the project requires a client library, how does it implement CLI and
  API bindings?

  Yes. No OSC transition yet.

Scorecard
---------

+---------------+
| Scorecard     |
+===============+
| N0_ |    Y    |
+---------------+
| N1_ |    N    |
+---------------+
| N2_ |    N    |
+---------------+
| N3_ |    Y    |
+---------------+
| N4_ |    Y    |
+---------------+
| N5_ |    N    |
+---------------+
| D1_ |    Y    |
+---------------+
| D2_ |    Y    |
+---------------+
| D3_ |    N    |
+---------------+
| D4_ |    Y    |
+---------------+
| C1_ |    N    |
+---------------+
| C2_ |    Y    |
+---------------+
| C3_ |    Y    |
+---------------+
| C4_ |    N    |
+---------------+
| C5_ |    Y    |
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

Final remarks: the subproject has made great progress lately. There are still
gaps to be filled, like lack of exhaustive coverage in the gate queue (current
tempest test is non-voting), and aligning with master. Client side extensions
will need to be rewritten in due course. However the subproject does not seem
to lack the resources and the focus to make timely progress when required.
