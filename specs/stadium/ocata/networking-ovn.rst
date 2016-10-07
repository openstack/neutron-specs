..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Networking-ovn Scorecard
========================

Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or relies on proprietary backends?

  networking-ovn implements a backend for the existing Neutron REST API.  It
  is not a client of Neutron REST APIs.  It also does not rely on any proprietary
  backends.

.. _N1:

* N1. Does the project integrate/use neutron-lib?

  Yes. The project uses neutron-lib with an overall migration status of 11.66% as of
  October 10, 2016.

.. _N2:

* N2. Do project members actively contribute to help neutron-lib achieve its
  goal?

  No. The project core team members are not actively contributing to neutron-lib.

.. _N3:

* N3. Do project members collaborate with the core team to enable subprojects
  to loosely integrate with the Neutron core platform by helping with the definition
  of modular interfaces?

  The project team members have helped in this area by opening bug reports and
  providing fixes as appropriate (e.g. https://bugs.launchpad.net/neutron/+bug/1597898).

.. _N4:

* N4. How does the project provide networking services? Does it use modular interfaces
  as provided by the core platform?

  The project uses the modular interfaces (ML2 and L3) provided by the core platform to
  enable networking services. Some of the project's networking services are provided via
  native services instead of via neutron agents.

.. _N5:

* N5. If the project provides new API extensions, have API extensions been discussed
  and accepted by the Neutron drivers team? Please provide links to API specs, if
  required.

  The project does not provide new API extensions.

Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (links to logs.openstack.org).

  Yes.

  * http://git.openstack.org/cgit/openstack/networking-ovn/tree/tox.ini
  * http://docs.openstack.org/developer/networking-ovn/

.. _D2:

* D2. If the project provide API extensions, does the project have an
  api-ref tox target, functional and continously working? Provide proof
  (links to logs.openstack.org).

  No.

.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continously working? Provide proof.

  Yes.

  * http://git.openstack.org/cgit/openstack/networking-ovn/tree/tox.ini
  * http://docs.openstack.org/releasenotes/networking-ovn/

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.

  The project itself provides primarily developer, administrator and
  deployer documention at http://docs.openstack.org/developer/networking-ovn/,
  and relies on community OpenStack neutron documentation for end user
  documentation.

Continuous Integration
----------------------

.. _C1:

* C1. Does the project have a Grafana dashboard showing historical trends of
  all the jobs available? Provide proof (links to grafana.openstack.org).

  Yes.

  * http://grafana.openstack.org/dashboard/db/networking-ovn-failure-rates

.. _C2:

* C2. Does the project have CI for unit coverage? Provide proof (links to
  logs.openstack.org)

  Yes.

  * http://status.openstack.org/openstack-health/#/g/project/openstack~2Fnetworking-ovn

.. _C3:

* C3. Does the project have CI for functional coverage? If so, does it include
  DB migration and sync validation?

  Yes. The project does not introduce any Data Model at the moment and hence it does
  not require DB migration and sync validation.

  * http://status.openstack.org/openstack-health/#/job/gate-networking-ovn-dsvm-functional

.. _C4:

* C4. Does the project have CI for fullstack coverage?

  No.

.. _C5:

* C5. Does the project have CI for Tempest coverage? If so, specify nature
  (API and/or Scenario).

  Yes. The jobs test both native and conventional networking services with both
  API and scenario tests as taken by the Tempest project.

  * http://status.openstack.org/openstack-health/#/g/project/openstack~2Fnetworking-ovn
  * http://git.openstack.org/cgit/openstack/networking-ovn/tree/devstack/devstackgatenativeservicesrc
  * http://git.openstack.org/cgit/openstack/networking-ovn/tree/devstack/devstackgaterc

.. _C6:

* C6. Does the project require CI for Grenade coverage?

  A non-voting job is available. More is needed.

  * http://logs.openstack.org/65/378065/7/check/gate-grenade-dsvm-networking-ovn-nv/06fda71/logs/
  * https://bugs.launchpad.net/networking-ovn/+bug/1464283
  * https://bugs.launchpad.net/networking-ovn/+bug/1627810

.. _C7:

* C7. Does the project provide multinode CI?

  This area is under active development. Some pointers below.

  * https://review.openstack.org/#/c/378032/
  * https://bugs.launchpad.net/networking-ovn/+bug/1621627
  * https://bugs.launchpad.net/networking-ovn/+bug/1634511

.. _C8:

* C8. Does the project support Python 3.x? Provide proof.

  Yes. Some pointers below.

  * http://logs.openstack.org/65/378065/7/check/gate-networking-ovn-python35/67049ca/


Release footprint
-----------------

.. _R1:

* R1. Does the project adopt semver?

  Yes.

.. _R2:

* R2. Does the project have release deliverables? Provide proof as available
  in the `release repo <http://git.openstack.org/cgit/openstack/releases/tree/>`_.

  Yes. The project has release deliverables with newton being the first.

  * http://git.openstack.org/cgit/openstack/releases/tree/deliverables/newton/networking-ovn.yaml

.. _R3:

* R3. Does the project use upper-constraints?

  Yes.

  * https://github.com/openstack/networking-ovn/blob/master/tox.ini#L8

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?

  Yes.

  * https://github.com/openstack/requirements/commit/c8f5860a70a7dd7b5a87ff4c88ffac0bd466c609


Stable backports
----------------

.. _S1:

* S1. Does the project have stable branches and/or tags? Provide history of
  backports.

  Yes. The project has stable branches with stable/newton being the first.

  * http://git.openstack.org/cgit/openstack/networking-ovn


Client library
--------------

.. _L1:

* L1. If the project requires a client library, how does it implement CLI and
  API bindings?

  The project does not require a client library.


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
| N3_ |    Y    |
+---------------+
| N4_ |    Y    |
+---------------+
| N5_ |    Y    |
+---------------+
| D1_ |    Y    |
+---------------+
| D2_ |    Y    |
+---------------+
| D3_ |    Y    |
+---------------+
| D4_ |    Y    |
+---------------+
| C1_ |    Y    |
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
| L1_ |    Y    |
+-----+---------+

Final remarks: networking-ovn is a well managed project. It would be nice if
members of the project would pay some of the neutron-lib tax for the wellbeing
of the wider Neutron community but one can only wish for so much. The project
lacks in area of upgrade and multinode testing, but the team is actively
working to fill these gaps.
