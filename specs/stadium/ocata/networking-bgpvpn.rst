..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Networking-bgpvpn Scorecard
===========================

Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or relies on proprietary backends?

  No. The project provides an API and Framework to interconnect BGP/MPLS VPNs to
  Openstack Neutron networks, routers and ports. It represents the server-side
  (plus its client-side mappings) to provide such a functionality. The server-side
  provides a driver-based mechanism to supply implementations of the API, which
  include an agent-based solution as available by the related project
  networking-bagpipe, as well as OpenDaylight and OpenContrail.

.. _N1:

* N1. Does the project integrate/use neutron-lib?

  Yes. So far the import ratio is ~15%. The neutron-lib periodic integration is
  available with periodic-networking-bgpvpn-py27-with-neutron-lib-master, though
  the switch to py35 has happened recently, so the project should switch too.

  * https://review.openstack.org/#/c/377738/

.. _N2:

* N2. Do project members actively contribute to help neutron-lib achieve its
  goal?

  No evidence of that.

.. _N3:

* N3. Do project members collaborate with the core team to enable subprojects
  to loosely integrate with the Neutron core platform by helping with the definition
  of modular interfaces?

  Team memembers have been particularly active in reviewing patches affecting
  the OVS pipeline.

.. _N4:

* N4. How does the project provide networking services? Does it use modular interfaces
  as provided by the core platform?

  The project adopts a service plugin model to plug into the API framework and provide
  driver extensions a la' ML2 pattern. The decomposition between bgpvpn and bagpipe
  especially in relation to where the L2 agent extension is located may need some
  rethinking.

.. _N5:

* N5. If the project provides new API extensions, have API extensions been discussed
  and accepted by the Neutron drivers team? Please provide links to API specs, if
  required.

  No.


Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (links to logs.openstack.org).

  Yes.

  * https://github.com/openstack/networking-bgpvpn/blob/master/tox.ini

.. _D2:

* D2. If the project provide API extensions, does the project have an
  api-ref tox target, functional and continously working? Provide proof
  (links to logs.openstack.org).

  No. There is some API documentation but it is not in the required format.

  * https://review.openstack.org/#/c/390196/

.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continously working? Provide proof.

  Yes.

  * http://docs.openstack.org/releasenotes/networking-bgpvpn/
  * https://github.com/openstack/networking-bgpvpn/blob/master/tox.ini

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.

  There is developer, and user documentation available.

  * http://docs.openstack.org/developer/networking-bgpvpn/


Continuous Integration
----------------------

.. _C1:

* C1. Does the project have a Grafana dashboard showing historical trends of
  all the jobs available? Provide proof (links to grafana.openstack.org).

  No.

  * https://review.openstack.org/#/c/385335/

.. _C2:

* C2. Does the project have CI for unit coverage? Provide proof (links to
  logs.openstack.org)

  Yes.

  * http://status.openstack.org/openstack-health/#/g/project/openstack~2Fnetworking-bgpvpn

.. _C3:

* C3. Does the project have CI for functional coverage? If so, does it include
  DB migration and sync validation?

  There is functional coverage, but it is experimental. DB migration and sync
  validation is provided via unit testing.

  * http://status.openstack.org/openstack-health/#/g/project/openstack~2Fnetworking-bgpvpn

.. _C4:

* C4. Does the project have CI for fullstack coverage?

  No.

.. _C5:

* C5. Does the project have CI for Tempest coverage? If so, specify nature
  (API and/or Scenario).

  Only API tests available.

  * https://github.com/openstack/networking-bgpvpn/tree/master/networking_bgpvpn_tempest/tests

.. _C6:

* C6. How does a project validate upgrades on a continuous basis? Does
  the project require or support CI for Grenade coverage?

  No Grenade testing is available.

.. _C7:

* C7. Does the project provide multinode CI?

  No.

.. _C8:

* C8. Does the project support Python 3.x? Provide proof.

  Yes.

  * http://logs.openstack.org/85/387885/16/experimental/gate-networking-bgpvpn-python35-db/5459830/testr_results.html.gz


Release footprint
-----------------

.. _R1:

* R1. Does the project adopt semver?

  Yes.

.. _R2:

* R2. Does the project have release deliverables? Provide proof as available
  in the `release repo <http://git.openstack.org/cgit/openstack/releases/tree/>`_.

  Yes, though the project notes do not seem to show up.

  * https://releases.openstack.org/newton/index.html
  * https://github.com/openstack/releases/blob/master/deliverables/_independent/networking-bgpvpn.yaml

.. _R3:

* R3. Does the project use upper-constraints?

  Yes.

  * https://github.com/openstack/networking-bgpvpn/blob/master/tox.ini#L8

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?

  Yes.

  * https://review.openstack.org/#/c/391505/

Stable backports
----------------

.. _S1:

* S1. Does the project have stable branches and/or tags? Provide history of
  backports.

  Yes. Stable liberty, mitaka and newton look aligned with Neutron's


Client library
--------------

.. _L1:

* L1. If the project requires a client library, how does it implement CLI and
  API bindings?

  There are client mappings but no OSC transition yet.

  * https://review.openstack.org/#/c/388870/


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
| N5_ |    N    |
+---------------+
| D1_ |    Y    |
+---------------+
| D2_ |    N    |
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

Final remarks: There are some gaps that need attention most notably API
documentation, client mappings and functional/scenario testing.
