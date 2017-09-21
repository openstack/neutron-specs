..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================
<Project> Scorecard
===================

This document template is meant to be used as a scorecard to assess how
a project eligible for inclusion meets the Neutron Stadium requirements
as defined in this `specification <http://specs.openstack.org/openstack/neutron-specs/specs/newton/neutron-stadium.html>`_.
If the outcome of the assessment is negative, the project inclusion is
rejected.


Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or rely on proprietary backends?

.. _N1:

* N1. Does the project integrate/use neutron-lib?

.. _N2:

* N2. Do project members actively contribute to help neutron-lib achieve its
  goal?

.. _N3:

* N3. Do project members collaborate with the core team to enable subprojects
  to loosely integrate with the Neutron core platform by helping with the definition
  of modular interfaces?

.. _N4:

* N4. How does the project provide networking services? Does it use modular interfaces
  as provided by the core platform?

.. _N5:

* N5. If the project provides new API extensions, have API extensions been discussed
  and accepted by the Neutron drivers team? Please provide links to API specs, if
  required.


Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (e.g. links to logs.openstack.org).

.. _D2:

* D2. If the project provides API extensions, does the project have an
  api-ref tox target, functional and continuously working? Provide proof
  (e.g. links to logs.openstack.org).

.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continuously working? Provide proof.

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.


Continuous Integration
----------------------

.. _C1:

* C1. Does the project have a Grafana dashboard showing historical trends of
  all the jobs available? Provide proof (links to grafana.openstack.org).

.. _C2:

* C2. Does the project have CI for unit coverage? Provide proof (links to
  logs.openstack.org).

.. _C3:

* C3. Does the project have CI for functional coverage? If so, does it include
  DB migration and sync validation?

.. _C4:

* C4. Does the project have CI for fullstack coverage?

.. _C5:

* C5. Does the project have CI for Tempest coverage? If so, specify nature
  (API and/or Scenario).

.. _C6:

* C6. How does a project validate upgrades on a continuous basis? Does
  the project require or support CI for Grenade coverage?

.. _C7:

* C7. Does the project provide multinode CI?

.. _C8:

* C8. Does the project support Python 3.x? Provide proof.


Release footprint
-----------------

.. _R1:

* R1. Does the project adopt `SemVer <http://semver.org/>`_?

.. _R2:

* R2. Does the project have release deliverables? Provide proof as available
  in the `release repo <http://git.openstack.org/cgit/openstack/releases/tree/>`_.

.. _R3:

* R3. Does the project use upper-constraints?

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?


Stable backports
----------------

.. _S1:

* S1. Does the project have stable branches and/or tags? Provide history of
  backports.


Client library
--------------

.. _L1:

* L1. If the project requires a client library, how does it implement CLI and
  API bindings?


Scorecard
---------

+---------------+
| Scorecard     |
+===============+
| N0_ |         |
+---------------+
| N1_ |         |
+---------------+
| N2_ |         |
+---------------+
| N3_ |         |
+---------------+
| N4_ |         |
+---------------+
| N5_ |         |
+---------------+
| D1_ |         |
+---------------+
| D2_ |         |
+---------------+
| D3_ |         |
+---------------+
| D4_ |         |
+---------------+
| C1_ |         |
+---------------+
| C2_ |         |
+---------------+
| C3_ |         |
+---------------+
| C4_ |         |
+---------------+
| C5_ |         |
+---------------+
| C6_ |         |
+---------------+
| C7_ |         |
+---------------+
| C8_ |         |
+---------------+
| R1_ |         |
+---------------+
| R2_ |         |
+---------------+
| R3_ |         |
+---------------+
| R4_ |         |
+---------------+
| S1_ |         |
+-----+---------+
| L1_ |         |
+-----+---------+

Final remarks: (To be compiled by PTL).
