..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add enable_new_agents to neutron server
==========================================

https://blueprints.launchpad.net/neutron/+spec/enable-new-agents

This proposal adds enable_new_agents config for operator maintenance to agents
in network node. It proposes a way that an agent can start without selectable
for auto-scheduling but manual-scheduling available so that a deployer can test
an agent manually.

Problem Description
===================

Neutron doesn't have a way to test a newly added network node by deploying test
resource before any customer resource on the node is deployed. When cloud
operators control network resources, they want to prevent users from
controlling the resources. For example, they may try to create test resources
on a new node while deploying the new node. Neutron can prevent users from
creating the resources on the node by agent's admin_state_up=False, but Neutron
cannot start agent with admin_state_up=False. Neutron always starts agent with
admin_state_up=True. Nova and Cinder have the setting of "enable_new_services"
in each conf to disable the initial service status to achieve this.

Neutron also has a problem for the maintenance scenario. Neutron can
prevent all users from creating the resources on the agent with
admin_state_up=False, but Neutron usually cannot allow admin user only
to create the resources. Currently, if
enable_services_on_agents_with_admin_state_down configuration
parameter is True, admin can create the resources on the agent.

Proposed Change
===============

This proposal adds enable_new_agents config for operator maintenance to agents
in network nodes. Neutron agent's admin_state_up is controlled by this config
while starting. The config is added to neutron-server since the proposal provides
all agents which have scheduling method(i.e.l3-agent, dhcp-agent, lbaas-agent)
and don't have the method(i.e. ovs-agent, metadata-agent) so that deployers
enable agents easily although this proposal gives a benefit to scheduling agents
only.

* Neutron agent starts with admin_state_up=True when enable_new_agents=True.
  This behaviour is default. User freely creates their resources on the agent.

* Neutron agent starts with admin_state_up=False when enable_new_agents=False.
  This behaviour is maintenance mode. In the case, user's resources cannot be
  created on the agent until admin changes admin_state_up to True.

The default value is True because this proposal doesn't intend to change a
traditional behaviour.

The proposal also presupposes that
enable_services_on_agents_with_admin_state_down is True in Neutron
servers.

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

None

Performance Impact
------------------

None

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

This proposal adds enable_new_agents to neutron.conf. This config is set True
as default value. It doesn't change a traditional behaviour of the agents.

Developer Impact
----------------

None

Community Impact
----------------

None

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Hirofumi Ichihara <ichihara-hirofumi>

Work Items
----------

* Add enable_new_agents config and the implements

Dependencies
============

None

Testing
=======

Tempest Tests
-------------

None

Functional Tests
----------------

Add tests which ensure agents start with admin_state_up False.

API Tests
---------

None

Documentation Impact
====================

User Documentation
------------------

The new config options will be documented.

The following is a maintenance scenario. The maintenance scenario will be
documented.

Precondition: Neutron servers run with
enable_services_on_agents_with_admin_state_down=True.

1. Admin prepares a new network node.
2. Admin adds enable_new_agents=False to neutron.conf and starts a neutron
   server, then their all agents.
3. All agents run with admin_state_up=False.
4. Admin needs to create a network (or router) and allocates it to a target
   agent.
5. Admin creates VM connected the network resources, then admin
   confirms the capability.
6. Admin deletes all resources and update agents to be
   admin_state_up=True after the test.

Developer Documentation
-----------------------

None

References
==========

* https://blueprints.launchpad.net/neutron/+spec/enable-new-agents
* http://lists.openstack.org/pipermail/openstack-operators/2015-March/006434.html
