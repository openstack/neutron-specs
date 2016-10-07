..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Networking-calico Scorecard
===========================

Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or relies on proprietary backends?

  No. The project implements Neutron plugins.

.. _N1:

* N1. Does the project integrate/use neutron-lib?

  No. Of the total ~200 total imports, neutron-lib is imported 0 times whereas
  Neutron is imported a few dozens. Some in flight, more needed.

  * https://review.openstack.org/#/c/387536/

.. _N2:

* N2. Do project members actively contribute to help neutron-lib achieve its
  goal?

  No.

.. _N3:

* N3. Do project members collaborate with the core team to enable subprojects
  to loosely integrate with the Neutron core platform by helping with the definition
  of modular interfaces?

  No, as the next point demonstrates.

.. _N4:

* N4. How does the project provide networking services? Does it use modular interfaces
  as provided by the core platform?

  The project has adopted some `questionable patterns <https://github.com/openstack/networking-calico/commit/2c302e6bafe5cfe564b9816aa2a67a3a65508807>`_.
  These patterns not only defeat the point of modularity but may lead to solutions that
  may be incredibly brittle. Team working to address this.

  * https://review.openstack.org/#/c/396427/

.. _N5:

* N5. If the project provides new API extensions, have API extensions been discussed
  and accepted by the Neutron drivers team? Please provide links to API specs, if
  required.

  It does not look like the project introduces new APIs.

Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (links to logs.openstack.org).

  Yes.

  * http://docs.openstack.org/developer/networking-calico/

.. _D2:

* D2. If the project provide API extensions, does the project have an
  api-ref tox target, functional and continously working? Provide proof
  (links to logs.openstack.org).

  No.

.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continously working? Provide proof.

  No.

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.

  There is some developer and deployer documentation. More being added.

  * https://review.openstack.org/#/c/388789/


Continuous Integration
----------------------

.. _C1:

* C1. Does the project have a Grafana dashboard showing historical trends of
  all the jobs available? Provide proof (links to grafana.openstack.org).

  No.

.. _C2:

* C2. Does the project have CI for unit coverage? Provide proof (links to
  logs.openstack.org)

  Yes. Though, it is puzzling why some test code is available under non-test
  modulee, `test_election <https://github.com/openstack/networking-calico/blob/d6c3dbf95f111827d63073150a719ea74a03c9ea/networking_calico/plugins/ml2/drivers/calico/test/test_election.py>`_ being an example.

  * http://logs.openstack.org/62/383462/2/check/gate-networking-calico-python27-ubuntu-xenial/9e979e7/

.. _C3:

* C3. Does the project have CI for functional coverage? If so, does it include
  DB migration and sync validation?

  No.

.. _C4:

* C4. Does the project have CI for fullstack coverage?

  No.

.. _C5:

* C5. Does the project have CI for Tempest coverage? If so, specify nature
  (API and/or Scenario).

  No, at the time of writing.

  * https://review.openstack.org/#/c/390908/
  * http://logs.openstack.org/34/361134/5/check/gate-tempest-dsvm-networking-calico-nv/1b4d652/logs/testr_results.html.gz.
  * https://review.openstack.org/#/c/392184/

.. _C6:

* C6. Does the project require CI for Grenade coverage?

  Potentially, but currently there is none.

.. _C7:

* C7. Does the project provide multinode CI?

  No.

.. _C8:

* C8. Does the project support Python 3.x? Provide proof.

  No. In progress.

  * https://review.openstack.org/#/c/365466/
  * https://review.openstack.org/#/c/365466/


Release footprint
-----------------

.. _R1:

* R1. Does the project adopt semver?

  Yes.

.. _R2:

* R2. Does the project have release deliverables? Provide proof as available
  in the `release repo <http://git.openstack.org/cgit/openstack/releases/tree/>`_.

  Yes.

  * https://github.com/openstack/releases/blob/master/deliverables/_independent/networking-calico.yaml

.. _R3:

* R3. Does the project use upper-constraints?

  No. In progress.

  * https://review.openstack.org/#/c/392165/
  * https://review.openstack.org/#/c/392168/

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?

  No. In progress.


Stable backports
----------------

.. _S1:

* S1. Does the project have stable branches and/or tags? Provide history of
  backports.

  The lack of stable branches and the fact that the code somewhat handles
  branching `in code <https://github.com/openstack/networking-calico/blob/master/networking_calico/plugins/ml2/drivers/calico/mech_calico.py#L52>`_
  is concerning.


Client library
--------------

.. _L1:

* L1. If the project requires a client library, how does it implement CLI and
  API bindings?

  No.


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
| N3_ |    N    |
+---------------+
| N4_ |    N    |
+---------------+
| N5_ |    N    |
+---------------+
| D1_ |    Y    |
+---------------+
| D2_ |    N    |
+---------------+
| D3_ |    N    |
+---------------+
| D4_ |    Y    |
+---------------+
| C1_ |    Y    |
+---------------+
| C2_ |    N    |
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
| C8_ |    N    |
+---------------+
| R1_ |    Y    |
+---------------+
| R2_ |    Y    |
+---------------+
| R3_ |    N    |
+---------------+
| R4_ |    N    |
+---------------+
| S1_ |    N    |
+-----+---------+
| L1_ |    N    |
+-----+---------+

Final remarks: the modularity and maturity of the networking-calico codebase is substantially
subpar compared to other Neutron subprojects. The absence of testing besides unit coverage is
problematic to say the least. Some of the patterns chosen to provide networking solutions are
an exemplification of the disconnect between the networking-calico and the Neutron core team.
Work required is not insurmountable but it takes time.
