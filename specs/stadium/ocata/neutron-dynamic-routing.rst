..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Neutron-dynamic-routing Scorecard
=================================

Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or relies on proprietary backends?

  The project implements its own set of Neutron API extensions on top of
  the Neutron core framework and it does so by using the service plugin model.
  The API exposed has open source implementations, and it provides a pluggable
  mechanism for other backends.

.. _N1:

* N1. Does the project integrate/use neutron-lib?

  Yes. The migration report shows that there are currently ~300 total imports.
  Neutron is imported ~80 times and Neutron-lib only ~10 times, for a migration
  percentage of 13.0400%

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

  The BGP extensions have been widely discussed and accepted.

  * http://specs.openstack.org/openstack/neutron-specs/specs/mitaka/bgp-dynamic-routing.html
  * https://github.com/openstack/neutron-dynamic-routing/tree/master/neutron_dynamic_routing/extensions


Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (links to logs.openstack.org).

  Yes.

  * https://github.com/openstack/neutron-dynamic-routing/blob/master/tox.ini
  * http://docs.openstack.org/developer/openstack-projects.html

.. _D2:

* D2. If the project provide API extensions, does the project have an
  api-ref tox target, functional and continously working? Provide proof
  (links to logs.openstack.org).

  No api-ref target, but the project has API documentation in tree. This
  should migrate over to neutron-lib's api-ref.

  * http://docs.openstack.org/developer/neutron-dynamic-routing/design/api.html


.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continously working? Provide proof.

  Yes. Though nothing has been release noted.

  * https://github.com/openstack/neutron-dynamic-routing/blob/master/tox.ini
  * http://docs.openstack.org/releasenotes/neutron-dynamic-routing/

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.

  Developer documentation is rather rich for a project of this age, user
  and deployer documentation is provided in the networking guide.

  * http://docs.openstack.org/developer/neutron-dynamic-routing/
  * http://docs.openstack.org/mitaka/networking-guide/adv-config-bgp-dynamic-routing.html


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

  * http://status.openstack.org/openstack-health/#/g/project/openstack~2Fneutron-dynamic-routing

.. _C3:

* C3. Does the project have CI for functional coverage? If so, does it include
  DB migration and sync validation?

  Yes, but it lacks DB migration and sync validation.

  * http://status.openstack.org/openstack-health/#/job/gate-neutron-dynamic-routing-dsvm-functional

.. _C4:

* C4. Does the project have CI for fullstack coverage?

  No.

.. _C5:

* C5. Does the project have CI for Tempest coverage? If so, specify nature
  (API and/or Scenario).

  Yes. Though it looks like there is only basic API coverage.

  * http://status.openstack.org/openstack-health/#/job/gate-neutron-dynamic-routing-dsvm-tempest

.. _C6:

* C6. Does the project require CI for Grenade coverage?

  Yes. But it has none.

.. _C7:

* C7. Does the project provide multinode CI?

  No.


.. _C8:

* C8. Does the project support Python 3.x? Provide proof.

  Yes.


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

  * https://github.com/openstack/neutron-dynamic-routing/blob/master/tox.ini#L11

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?

  Yes.

  * https://github.com/openstack/requirements/commit/d722a8add81bb20943564146904e202dda85a526


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

  There are client extensions. It looks like OSC transition has stalled badly.

  * https://review.openstack.org/#/c/340763/


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


Final remarks
-------------

Since the project setup at the beginning of Newton, lots has been achieved.
There are some gaps that need to be filled, and a focussed team should have
no problem on achieving those in time of the Ocata-1 deadline.
