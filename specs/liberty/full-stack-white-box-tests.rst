..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Neutron/Full-Stack White-Box Tests Framework
==============================================

Launchpad blueprint:

 https://blueprints.launchpad.net/neutron/+spec/integration-tests

This blueprint specifies the framework needed to write new full-stack tests
targeting Neutron components (as opposed to tests which validate integration
among different OpenStack components). These tests are intended to be
white-box - they will validate Neutron's functionality, mainly on the operating
system level.

In other words, full-stack tests intend to test Neutron on its own, without
any other OpenStack components, and fall between functional and Tempest
tests.

Problem Description
===================

Currently there are 3 main types of tests for Neutron:

1. Unit tests, which intend to check the code mostly at the function level
   where each function is tested separately. For example: make sure that
   function X returns value Y when the arguments are A, B and C.
2. Functional tests, which target only single sub-components and intend to
   validate interaction with external (system or host-local) resources
   (mostly operating system interactions and outside packages like Open-vSwitch
   and iproute2...) An example functional test could look like:

.. code-block:: python

  def test_router_lifecycle(self):
      """Create a router and check that the agent creates various resources.

      self.agent is an instance of type L3NATAgent (the main class in the L3
      Agent) and is created as part of the test. The assertions make sure that
      various OS resources (like interfaces, namespaces, processes) are created
      as part of the creation of a new router.
      """
      router_info = self.generate_router_info(enable_ha)
      router = self.manage_router(self.agent, router_info)

      self.assertTrue(self._namespace_exists(router))
      self.assertTrue(self._metadata_proxy_exists(self.agent.conf, router))
      self._assert_internal_devices(router)
      self._assert_external_device(router)
      self._assert_gateway(router)
      self._assert_floating_ips(router)
      self._assert_snat_chains(router)
      self._assert_floating_ip_chains(router)

      self._delete_router(self.agent, router.router_id)
      self._assert_router_does_not_exist(router)


3. Tempest tests, which check that OpenStack as a whole and Neutron in
   particular behave as intended. Tempest tests are split to 2 groups:


   a. Scenario tests (start a new instance and make sure that the instance has
      connectivity to the external network). These tests often intend to check
      more than one component at a time (as opposed to full-stack tests
      discussed here, which check only Neutron).
   b. API tests (create a new router, make sure that the controller is aware of
      it and delete it).

There is currently no way to check Neutron as a standalone project and the
interaction between the different Neutron components without other OpenStack
components such as Nova or Cinder.

Proposed Change
===============

The framework will be written in the Neutron tree, thus allowing for easier
writing of new tests (as opposed to out-of-tree Tempest tests), being aware of
Neutron resources used during tests (executing Neutron agents and cleaning up
used resources), and running specific Neutron tests intended to check the
interaction between the different Neutron components.

The framework intends to achieve state isolation, achieved through reseting the
state of Neutron components and restarting Neutron agents when needed (Tempest
currently does this manually - routers created during tests needs to be
explicitly deleted afterwards).

Currently the intention is to reset said state after each full-stack test, but
this can be easily changed (we might want to do so after each test suite
instead) once enough tests are added to the framework which will allow reliable
benchmarking.

The framework will support:

1. Writing full-stack tests using the unittest module,
2. Restarting all the Neutron processes which are needed (neutron-server,
   L3 agent, etc...) using setUp and cleanUp when appropriate,
3. Wiping the databases and cleaning up resources (specifically running
   neutron-ovs-cleanup and neutron-netns-cleanup),
4. Using only python-neutronclient to communicate with Neutron.

.. code-block:: python

  def test_ha_router_startup(self):
      """Create an HA router for two agents and make sure they can communicate.

      While this full-stack test look similar to 'test_router_lifecycle', it
      relies on the Neutron API (python-neutronclient) to create a new router
      and actual processes (one neutron-server, 2 pairs of L3 and OVS agents),
      meaning that once a request is made, it goes through all relevant agents
      and components (instead of a single sub-component). OS resources are also
      checked in this test.
      """
    def test_ha_router_startup(self):
        l3_agents = ['neutron-l3-1-agent', 'neutron-l3-2-agent']

        router = self.client.create_router(
            body={'router': {'name': 'router-test',
                             'tenant_id': uuidutils.generate_uuid(),
                             'ha': True}})
        router_id = router['router']['id']

        self._assert_namespaces_exists(router_id, l3_agents)
        master, slave = self._get_master_and_slave(router_id, l3_agents)

        # Only if the two agents can ping each other will the other router
        # change itself to be the backup router.
        test_daemon.wait_until(
            lambda: 'backup' in self._get_ha_state(router_id, slave))


Running full-stack tests will require one to setup devstack and stop all the
services ('./stack.sh', './unstack.sh'), then using tox to run the tests.
Further steps might be needed down the line to setup databases and other
resources, but an attempt to minimize the steps required to run the tests will
be taken over time.

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Users will be able to manually execute full-stack tests using normal tools
(such as tox), though admins and cloud users don't need to run these tests
after deployment.

No changes to python-neutronclient are needed.

Performance Impact
------------------

None

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Writing new features might require writing full-stack tests for them, as well
as unit and function tests.

Community Impact
----------------

Running full-stack tests will be available to run, similarly to current unit
and functional tests, and will increase the level of credibility and
reliability of new and existing Neutron features (both upstream and
downstream).

Alternatives
------------

Writing these kind of tests could be done in Tempest, however this approach was
not chosen for several reasons:

1. The direction of the QA effort in OpenStack precludes performing these kind
   of tests in Tempest since it is refocusing on multi-service testing.
2. Full-stack tests allow for greater flexibility when developers write new
   Neutron code. Tempest policy dictates that only API tests are used with the
   various components, thus disallowing checks which involve system interaction
   and limiting tests of some of the features mentioned previously. Since the
   tests will be placed in-tree, the policy regarding new tests will be easier
   to determine.
3. With this new type of testing we could test specific conditions (like
   consistency and sustained connectivity across agent restarts), which is
   impossible at this moment with Tempest.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

1. John Schwarz <jschwarz@redhat.com>

Other contributors:

1. Assaf Muller <amuller@redhat.com>
2. Maru Newby <marun@redhat.com>

Work Items
----------

1. Add ability to manage daemonized Neutron processes,
2. Implement supporting full-stack white-box testing framework,
3. Add an example of a full-stack test.

Dependencies
============

1. https://review.openstack.org/#/c/124136/
2. https://review.openstack.org/#/c/125109/

Testing
=======

The ultimate goal is having these new kinds of full-stack tests be run as part
of the gate - coders will have to verify that their code passes specific
'full-stack white-box tests' gate job in order to get their patches merged.

Furthermore, changes to the infra project will be required to add these kind of
tests to the gate (first as a non-voting and later as a voting gate job).

Tempest Tests
-------------

None

Functional Tests
----------------

None

API Tests
---------

None

Documentation Impact
====================

User Documentation
------------------

None

Developer Documentation
-----------------------

The following wiki page contains documentations and "best practices" regarding
in-tree tests and full-stack tests specifically. Before writing new
tests, a developer should read the following wiki page:
https://wiki.openstack.org/wiki/Neutron/InTreeTests

TESTING.rst should be updated to reflect the new tests framework.

References
==========

None
