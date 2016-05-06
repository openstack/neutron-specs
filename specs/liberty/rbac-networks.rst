..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Role-based Access Control for Networks
======================================

URL of launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/rbac-networks

We need the ability to share Neutron networks to subsets of tenants instead
of the all-or-nothing choice we have now. This will enable many more network
management workflows encountered in enterprise/private-cloud deployments.

This will be achieved through a new 'role-based access control' table with
entries that contain an object UUID, an object type, a target tenant, an
action and a tenant ID representing the owner of the policy.
In this generic format, it should be relatively easy to extend
role-based access control to other Neutron objects and actions.


Problem Description
===================

The current option to share a Neutron network is binary. Networks are either
shared to every tenant or are not shared at all. This eliminates a lot of
grouping concepts that show up in 'legacy' network configurations where
classes of devices or users can access a network while others cannot.
While this is okay for many public cloud deployments where the network
abstractions serve only as L2 isolation tools, it fails to fulfill many
private cloud use cases.

For example, there is no way for an administrator to define a network that
has access to a restricted resource (e.g. a multicast/broadcast feed of data)
and only allow certain tenants to attach servers to it. The same applies
for networks that may have special auditing requirements, security controls,
service guaruntees, floating IP pools, etc.

In a similar context, a cloud administrator may want to completely eliminate
the ability for most tenants to create networks, subnets, and routers and
just have them attach to pre-created networks corresponding to their department
in an organization.


Proposed Change
===============

Introduce a new RBAC table that will control sharing of Neutron networks
between tenants. Also introduce a new API to work with RBAC entries that will
be generic enough to easily be extended to other resources that people want
controlled by RBAC.

The logic surrounding shared networks will then be re-written to leverage the
new RBAC code as the first implementation. The current 'shared' attribute on
the network will be mapped into a wildcard entry in the new table to preserve
backwards compatibility (more details below).

The forwarding semantics and ownership of objects will not be impacted by this
specification. This specification only impacts which users are allowed to
attach to networks - not how they attach or what happens afterwards.

To keep things simple, this will be a whitelist-only model to begin with.
i.e. the action column can't be a negative that states a certain tenant
can't do something.

Tenants will be able to delete ports on networks they own even if the port does
not belong to them. This is to allow tenants to have final say over networks
they own.


Data Model Impact
-----------------

Each RBAC entry will contain the following important pieces of information:
the object identifier, the object type, the tenant identifier, and the
allowed action. All entries are to allow the action they describe. There
won't be any 'deny' rules at this time.

The object ID is simply the UUID of the object to which the RBAC entry will be
applied.

The object type simply describes the type of object to which the object ID
is a reference. For this specification, it will always be 'network' since
that's the only object being introduced into RBAC right now. This information
is redundant since the object ID is unique; however, this will allow us to have
relationship tables for each type we want to support. These allow the cleaning up
of RBAC entries on an object deletion to be left to the database CASCADE
functionality. The object type isn't stored in the database, it's used to
determine which table to use.

The target tenant will be the tenant (a.k.a. project) ID to which the RBAC entry
is granting permission to perform an action on the target object. This entry
may also be an asterisk to represent that it applies to all tenants.

The action will describe what the tenant can do with the object.
For the sake of this specification, we will only have one action to start,
which will be something like 'access_as_shared' to indicate that the tenant
can access it like it would a shared network. This could then easily be
extended in the future to include something like 'access_as_external' to
provide RBAC for external networks as well. Actions will be discoverable
for each type via the API.

Finally, each RBAC entry will also have a tenant ID that indicates the tenant
that created the policy itself. This will normally be the same tenant as the
object being shared, but it might not be in the case of admin-created policies
on other tenant's networks.


DB Tables
---------

Each object type that RBAC applies to will get its own RBAC table.
The unique constraint will only be across all of the API fields. (i.e. Each
tenant could have multiple entries per object with different actions).
So for this spec there will just be one new networkrbacs table.

The 'shared' column will be removed from the networks table and that will be
replaced with a query to the new table for the presence of the '*' for that
network with the 'access_as_shared' action.



REST API Impact
---------------

These RBAC entries will be enabled at a new API endpoint ('/rbac-policies').
In order to create an entry for an object, the tenant must be the owner
of that object.

In order to create an entry with a wildcard membership
tenants, the request will have to be performed in an admin context by
default but can be enabled for all tenants via policy.json.
This is done to prevent tenants from polluting every other
tenant's network list by creating wildcard entries.

In the future we can revisit to add a 'reshare' action or something similar if
we encounter a use case where the object owner wants to give the ability to
another tenant to share the object to more tenants.

+-------------+-------+---------+---------+------------+----------------+
|Attribute    |Type   |Access   |Default  |Validation/ |Description     |
|Name         |       |         |Value    |Conversion  |                |
+=============+=======+=========+=========+============+================+
|id           |string |R        | auto    |            |id of ACL entry |
|             |(UUID) |         |         |            |                |
+-------------+-------+---------+---------+------------+----------------+
|tenant_id    |string |R        | auto    |            |owner of ACL    |
|             |(UUID) |         |         |            |entry           |
+-------------+-------+---------+---------+------------+----------------+
|object_id    |string |RW       |N/A      |object      |object          |
|             |(UUID) |         |         |exists      |affected by ACL |
+-------------+-------+---------+---------+------------+----------------+
|object_type  |string |RW       |N/A      |type has    |type of object  |
|             |       |         |         |rbac table  |                |
+-------------+-------+---------+---------+------------+----------------+
|target_tenant|string |RWU      |*        |string      |tenant ID the   |
|             |       |         |         |            |entry affects.  |
|             |       |         |         |            |* for all       |
+-------------+-------+---------+---------+------------+----------------+
|action       |string |RW       |N/A      |in actions  |allowed tenant  |
|             |       |         |         |for object  |action on object|
+-------------+-------+---------+---------+------------+----------------+



Example Entries
---------------

A legacy shared network:
* object_id = <some_net_id>
* object_type = network
* target_tenant = *
* action = 'access_as_shared'
* id = <uuid>  # auto generated
* tenant_id = <uuid-of-policy-creator>  # generated by API

A network shared to a specific tenant:
* object_id = <some_net_id>
* object_type = network
* target_tenant = <some_tenant_id>
* action = 'access_as_shared'
* id = <uuid>  # auto generated
* tenant_id = <uuid-of-policy-creator>  # generated by API


Security Impact
---------------

Tenants will be able to share networks with each other. This shouldn't be a
major issue since the ownership will never change so they will still have to
take responsibility for them when it comes to bandwidth accounting and incident
responses.

If a user knows the tenant ID of someone they want to attack, they could share
networks with that tenant to pollute their network list with entries that may
be overwhelming or that may have names that trick the target tenant into
attaching VMs to it.

This can likely be solved with a choice of default filtering for networks.
Unless requested by the user, the client can filter out shared networks. Then
a UI like Horizon could display the shared networks with a flag next to them
to distinguish them from the tenant's networks. Thoughts?


Notifications Impact
--------------------

N/A


Other End User Impact
---------------------
New CLI workflow for setting these permissions:

* neutron rbac-create <net-uuid|net-name> --type network --target-tenant <tenant-uuid> --action access_as_shared

Update:
* neutron rbac-update <rbac-uuid> --target-tenant <other-tenant-uuid>

List entries:

* neutron rbac-list

Show entry:

* neutron rbac-show <object-id>

Deleting:

* neutron rbac-delete <rbac-uuid>

List available actions:

* neutron rbac-list-actions <object-type>

There should be no impact to the regular global shared network workflow. The
new API usage will only be required for fine-grained entries.

From the perspective of a tenant that has a network shared to it, the network
will show up as 'shared' just like a globally shared network would.


Performance Impact
------------------

Checking the shared attribute for the network will now involve a join to
another table. The impact to network listing will be quantified during code
review.


Other Deployer Impact
---------------------

N/A


Developer Impact
----------------

N/A


Community Impact
----------------

This change shouldn't impact the community in any major way. It will
introduce a new method of restricted sharing, but there isn't anything major
that should hit out-of-tree drivers/plugins.

Alternatives
------------

N/A


IPv6 Impact
-----------

N/A


Implementation
==============

Assignee(s)
-----------
kevinbenton
sballe


Work Items
----------
* Add the DB model, REST API changes, UTs to the Neutron server
* Adjust existing 'shared' attribute to use rbac and add migration script
* Update the client to CRUD the ACLs
* Add API tests


Dependencies
============

N/A


Testing
=======

Tempest Tests
-------------

N/A. API tests should be sufficient


Functional Tests
----------------

No functional test is likely necessary for this work. All of this is
at the API layer without impacting the dataplane.


API Tests
---------

* Excercise basic CRUD of ACL entries
* Make sure networks are revealed and hidden as ACL entries are changed
* Delete port of another tenant from tenant with shared network


Documentation Impact
====================


User Documentation
------------------

The workflow for adding RBAC entries will need to be added.
The workflow for a normal shared network should be the same
so existing docs shouldn't need to be changed.


Developer Documentation
-----------------------

The new sharing API will need to be documented.


References
==========

Here is a dragon breathing fire:

\
_\_(')<~~~
\____)
