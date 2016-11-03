..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Networking-onos Scorecard
=========================

Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or relies on proprietary backends?

  No, the project maps the various Neutron APIs on top of the ONOS SDN controller.

.. _N1:

* N1. Does the project integrate/use neutron-lib?

  No. The project imports Neutron only a couple of dozen times. That said,
  neutron-lib looks to be included in the requirements.

.. _N2:

* N2. Do project members actively contribute to help neutron-lib achieve its
  goal?

  No.

.. _N3:

* N3. Do project members collaborate with the core team to enable subprojects
  to loosely integrate with the Neutron core platform by helping with the definition
  of modular interfaces?

  There is no evidence of that.

.. _N4:

* N4. How does the project provide networking services? Does it use modular interfaces
  as provided by the core platform?

  Yes, though there are some `code smells <https://github.com/openstack/networking-onos/commit/d6b982c37c3809157b9265cf6000cf6735007b80>`_ that may need clean up.

.. _N5:

* N5. If the project provides new API extensions, have API extensions been discussed
  and accepted by the Neutron drivers team? Please provide links to API specs, if
  required.

  The project currently provides a driver for the SFC API (though the requirement seems
  `misplaced <https://github.com/openstack/networking-onos/blob/9df5b664deeada9557183c03189ce6ee40d81a5f/test-requirements.txt>`_.


Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (links to logs.openstack.org).

  Yes.

  * http://docs.openstack.org/developer/networking-onos/

.. _D2:

* D2. If the project provide API extensions, does the project have an
  api-ref tox target, functional and continously working? Provide proof
  (links to logs.openstack.org).

  The project does not propose new APIs.

.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continously working? Provide proof.

  The target seems set up but no release notes are available.

  * https://review.openstack.org/#/c/347209/

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.

  The documentation is available but content is bare bone.


Continuous Integration
----------------------

.. _C1:

* C1. Does the project have a Grafana dashboard showing historical trends of
  all the jobs available? Provide proof (links to grafana.openstack.org).

  No.

.. _C2:

* C2. Does the project have CI for unit coverage? Provide proof (links to
  logs.openstack.org)

  Yes. Broken at the time of writing.

  * http://logs.openstack.org/91/361491/3/check/gate-networking-onos-python27-ubuntu-xenial/7e1a4a5/testr_results.html.gz

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

  Yes, but non-voting and broken.

  * http://logs.openstack.org/91/361491/3/check/gate-tempest-dsvm-networking-onos/81dae2c/logs

.. _C6:

* C6. Does the project require CI for Grenade coverage?

  That does not seem like a Grenade style job be required, but there is no
  hint that the plugins would be able to talk to controllers running with
  different versions.

.. _C7:

* C7. Does the project provide multinode CI?

  No.


.. _C8:

* C8. Does the project support Python 3.x? Provide proof.

  No.


Release footprint
-----------------

.. _R1:

* R1. Does the project adopt semver?

  Yes.

.. _R2:

* R2. Does the project have release deliverables? Provide proof as available
  in the `release repo <http://git.openstack.org/cgit/openstack/releases/tree/>`_.

  Yes.

  * https://github.com/openstack/releases/blob/master/deliverables/_independent/networking-onos.yaml

.. _R3:

* R3. Does the project use upper-constraints?

  Yes.

  * https://github.com/openstack/networking-onos/blob/master/tox.ini#L11

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?

  Yes.

  * https://github.com/openstack/requirements/commit/0fabc239e2a9d5453e17ac18b8b4b6c22412cfb0


Stable backports
----------------

.. _S1:

* S1. Does the project have stable branches and/or tags? Provide history of
  backports.

  At the time of writing there is a stable mitaka branch, with no substantial
  history.

  * https://review.openstack.org/#/q/project:openstack/networking-onos+branch:stable/mitaka


Client library
--------------

.. _L1:

* L1. If the project requires a client library, how does it implement CLI and
  API bindings?

  It does not seem like client extensions are required.


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
| N4_ |    Y    |
+---------------+
| N5_ |    Y    |
+---------------+
| D1_ |    Y    |
+---------------+
| D2_ |    Y    |
+---------------+
| D3_ |    N    |
+---------------+
| D4_ |    N    |
+---------------+
| C1_ |    N    |
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
| R3_ |    Y    |
+---------------+
| R4_ |    Y    |
+---------------+
| S1_ |    N    |
+-----+---------+
| L1_ |    Y    |
+-----+---------+

Final remarks: the networking-onos project is not well managed, it lacks in
many areas and is considerably subpar compared to other Neutron subprojects.
Steering it in the right direction in time of the Ocata-1 deadline would
require an herculean effort. That said, at the time of writing (October
2016), the project has not seen active development since the end of August
2016. The project will be removed from the Stadium for the Ocata release.
