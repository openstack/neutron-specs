..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Clean up resources when a tenant is deleted
===========================================
https://blueprints.launchpad.net/neutron/+spec/tenant-delete

Problem description
===================
OpenStack projects currently do not delete tenant resources when a tenant
is deleted. For example, a user registers to a public cloud, and a script
auto generates a Keystone user and tenant, along with a Neutron router and
a tenant network. Later, the user changes his mind and removes his subscription
from the service. An admin deletes the tenant, yet the router and network
remain, with a tenant id of a tenant that no longer exists. This issue can
cause ballooning databases and operational issues for long-standing clouds.

Proposed change
===============
1) Expose an admin CLI tool that either accepts a tenant-id and deletes its
   resources, or finds all tenants with left-over resources and deletes them.
   It does this by listing tenants in Keystone and deletes any resources
   not belonging to those tenants. The tool will support generating a JSON
   document that details all to-be-deleted tenants and their resources,
   as well as a command to delete said tenants.
2) The ability to configure Neutron to listen and react to Keystone tenant
   deletion events, by sending a delete on each orphaned resource.

The Big Picture
---------------
Solving the issue only in Neutron is only the beginning. The goal of this
blueprint is to implement the needed changes in Neutron, so the implementation
may serve as a reference for future work and discussion for an OpenStack-
wide solution.

I aim to lead a discussion about this issue in the K summit with leads
from other projects.

Alternatives
------------
Should the events be reacted upon? According to the principle of least surprise,
I think an admin would expect tenant resources to be deleted when he deletes a
tenant, and not have to invoke a CLI tool manually, or set up a cron job.

Currently Keystone does not emit notifications by default, which poses a
challenge. I'll try to change the default in Keystone, as well as make Neutron
listen and react by default.

How would an OpenStack-wide solution work? Would each project expose its
own CLI utility, which an admin would have to invoke one by one? Or would
projects simply listen to Keystone events? Another alternative would be
that each project would implement an API. I argue that it introduces higher
coupling for no significant gain.

Data model impact
-----------------
None

REST API impact
---------------
None

Security impact
---------------
Neutron will now be willing to accept RPC messages in a well known format,
deleting user data.

Notifications impact
--------------------
Configuration changes to listen to the same exchange and topic as Keystone
notifications.

Other end user impact
---------------------
See documentation section.

Performance Impact
------------------
Deleting all tenant resources is a costly operation and should probably
be turned off for scale purposes. A cron job can be setup to perform clean
up during off hours.

Other deployer impact
---------------------
| Keystone needs to be configured to emit notifications via (Non-default):
| notification_driver = messaging

| The following keys need to match in keystone.conf in the general section,
  and neutron.conf in the [identity] section:
| control_exchange = openstack
| notification_topics = notifications

They match by default.

The new neutron-delete-tenant executable may be configured as a cron job.

Developer impact
----------------
None

Implementation
==============

Assignee(s)
-----------
Assaf Muller

Work Items
----------
1) Implement a CLI tool that detects "orphaned" tenants and deletes all of
   their resources.
2) Make Neutron listen to Keystone notifications.
3) Implement a callback that accepts a tenant id and deletes all of its
   resources. This will be implemented as a new service plugin. It will reuse
   code from step 1.

Note that Neutron has intra-dependencies. For example, you can not delete
a network with active ports. This means that resources need to be deleted
in specific order. Shared resources need to be dealt with in a special manner:

If there are multiple tenants that need to be deleted, (IE: More than
one orphaned tenant), the solution is to delete resources breadth first,
and not tenant by tenant. IE: First delete all ports by all tenants,
then all networks.

An issue remains if the admin tries to delete a tenant with a shared network
without first deleting another tenant with ports belonging to the shared
network. The non-shared resources will be successfully deleted but the shared
network will not. At this point the first tenant will have a network which
was not cleaned up and has no reason to be deleted, unless at some point in the
future the active ports will be deleted, then the CLI tool may be used with
the --automatic flag. I'm fine with this deficiency.

Dependencies
============
I want to make Keystone emit notifications by default.

Testing
=======
I think unit tests are not the correct level to test this feature.
What we need is both functional and integration tests.

Functional tests are currently proposed as "unit" tests: Like a significant
amount of currently implemented unit tests, these are in fact functional tests,
making actual API calls to an in-memory DB. This is the current proposal
for the Neutron side of things: Make API calls to create resources under a
certain tenant, import the Identity service plugin and "delete" the tenant.
At this point assertions will be made to verify that the previously created
resources are now gone.

Maru has been leading an effort [1] to move these type of tests to the
functional tree and CI job, as well as make calls directly to the plugins and
not via the API. There remains an open question if this blueprint should depend
on Maru's, placing more focus on Maru's efforts, which is missing a framework
that enables calls to the plugins for all types of resources.

What would still be missing is coverage for the Keystone to Neutron
notifications, and this would obviously belong to Tempest. This would
require a configuration change to the gate, as Keystone does not emit
notifications by default.

Finally, I propose that the --automatic functionality of the CLI tool would
be tested in Neutron, by mocking out the call to Keystone.
Reminder: The CLI tool makes an API call to get a Keystone tenant list, then
goes through the resources in Neutron, deleting orphaned resources.
The mock would return a predetermined list of tenants. Note: I propose
to test the 'automatic' functionality of the Identity service plugin directly,
not via the CLI tool.

Documentation Impact
====================
There will be new configuration options in neutron.conf, as well as a new
CLI tool to delete left over resources.

| neutron.conf:
| [identity]
| control_exchange = openstack
| notification_topics = notifications

| CLI tool:
| neutron-delete-tenant --tenant-id=<tenant_id> --automatic --generate --delete

| --tenant-id will delete all resources by <tenant_id>.
| --generate reports a JSON report of all tenants not currently existing in
| Keystone, and their resources. The report will be placed in $STATE_PATH/
| identity.
| --delete will delete all resources listed in $STATE_PATH/identity.
| --automatic performs --generate and then --delete.

--tenant-id is exclusive with the other three options.

References
==========
[1] https://blueprints.launchpad.net/neutron/+spec/neutron-testing-refactor

