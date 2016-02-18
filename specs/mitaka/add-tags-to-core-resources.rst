..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Add Tags to Neutron Resources
==============================================

https://blueprints.launchpad.net/neutron/+spec/add-tags-to-core-resources

Neutron resources in current DB model do not contain any tags, and
don't have a generic consistent way to add tags or/and any other data
by the user.
Tagging resources can be used by external systems or any other
clients of the Neutron REST API (and NOT backend drivers).

The following use cases refer to adding tags to networks, but the same
can be applicable to any other Neutron resource:

1) Ability to map different networks in different OpenStack locations
   to one logically same network (for Multi site OpenStack)

2) Ability to map Id's from different management/orchestration systems to
   OpenStack networks in mixed environments, for example for project Kuryr,
   map docker network id to neutron network id

3) Leverage tags by deployment tools

4) allow operators to tag information about provider networks
   (e.g. high-bandwith, low-latency, etc)

5) new features like get-me-a-network or a similar port scheduler
   could choose a network for a port based on tags

Problem Description
===================

In most popular REST API interfaces, objects in the domain model can be
"tagged" with zero or more simple strings. These strings may then be used
to group and categorize objects in the domain model.

In order to align Neutron's REST API with the Internet's common understanding
of `resource tagging`_, we can add an API that allows normal users
to add, remove and list tags for a neutron resource.


Proposed Change
===============

This spec is for adding a new way for *API users* to be able to tag
their neutron resources with simple strings.

Add an API that allows a user to add, remove, and list tags for a resource.

Similar to the approved nova spec `tag instances`_, and in accordance to
these `guidelines`_.

This spec proposes to support adding tags to Neutron resources which
are not meant to be interpreted by any specific backend implementation.
In the proposed implementation tags will only be visible in the API level and
not be available in the plugin level.

Data Model Impact
-----------------

The tag is an opaque string and is not intended to be interpreted or
even read by the Neutron backends.

For the database schema, the following table constructs would suffice::

    CREATE TABLE tags (
        standard_attribute_id BIGINT NOT NULL FOREIGN KEY,
        tag VARCHAR(60) NOT NULL CHARACTER SET utf8
         COLLATION utf8_ci PRIMARY KEY,
    );


For example adding a tag "blue" to network with standard_attribute_id 16 will
introduce the following entry:

 standard_attribute_id : 16
 tag: "blue"

REST API examples can be found in the next section.

In this table schema the tag is a string value attached to this resource.

Neutron implementation now have a common resource object for every type which
is saved in the DB. view `common neutron object`_ for more details.
standard_attribute_id reference the Neutron resource entry in that common table
for every tag entry we add.
Adding this references in the entry can help us keep data integrity
and make sure that all references for a specific resource are deleted
when the resource itself is deleted.

Authorization to manage the tag equates with that for the resource which the
tag is attached.

REST API Impact
---------------

The tag CRUD operations API would look like the following:

A list of tags for the specified network returns with the network details
information ::

    GET /v2.0/networks/{network_id}

Response ::

    {
        'id': {network_id},

        ... other network resource properties ...

        'tags': ['foo', 'bar', 'baz']
    }

Get **only** a list of tags for the specified network ::

    GET /V2.0/networks/{network_id}.json?fields=tags

Response ::

    {
        'tags': ['foo', 'bar', 'baz']
    }

Replace set of tags on a network ::

    PUT /v2.0/networks/{network_id}/tags

with request payload ::

    {
        'tags': ['foo', 'bar', 'baz']
    }

Response ::

    {
        'tags': ['foo', 'bar', 'baz']
    }

If the number of tags exceeds the limit of tags per network, shall return
a `400 Bad Request`
(tags limit is either hard coded or configurable in neutron conf file)

Add a single tag on a network ::

    PUT /v2.0/networks/{network_id}/tags/{tag}

Returns `201 Created`.

If the tag already exists, no error is raised, it just returns the
`409 Conflict`

Check if a tag exists or not on a network ::

    GET /v2.0/networks/{network_id}/tags/{tag}

Returns `204 No Content` if tag exist on a network.

Returns `404 Not Found` if tag doesn't exist on a network.

Remove a single tag on a network ::

    DELETE /v2.0/networks/{network_id}/tags/{tag}

Returns `204 No Content` upon success. Returns a `404 Not Found` if you
attempt to delete a tag that does not exist.

Remove all tags on a network ::

    DELETE /v2.0/networks/{network_id}/tags

Returns `204 No Content`.

The API that would allow searching/filtering of the `GET /v2.0/networks`
REST API call would add the following query parameters:

* `tags`
* `tags-any`
* `not-tags`
* `not-tags-any`

To request the list of networks that have a single tag, ``tags`` argument
should be set to the desired tag name. Example::

    GET /v2.0/networks?tags=red

To request the list of networks that have two or more tags, the ``tags``
argument should be set to the list of tags, separated by commas. In this
situation the tags given must all be present for a network to be included in
the query result. Example that returns networks that have the "red" and "blue"
tags::

    GET /v2.0/networks?tags=red,blue

To request the list of networks that have one or more of a list of given tags,
the ``tags-any`` argument should be set to the list of tags, separated by
commas. In this situation as long as one of the given tags is present the
network will be included in the query result. Example that returns the networks
that have the "red" or the "blue" tag::

    GET /v2.0/networks?tags-any=red,blue

To request the list of networks that do not have one or more tags, the
``not-tags`` argument should be set to the list of tags, separated by commas.
In this situation only the networks that do not have any of the given tags will
be included in the query results. Example that returns the networks that do not
have the "red" nor the "blue" tag::

    GET /v2.0/networks?not-tags=red,blue

To request the list of networks that do not have at least one of a list of
tags, the ``not-tags-any`` argument should be set to the list of tags,
separated by commas. In this situation only the networks that do not have at
least one of the given tags will be included in the query result. Example that
returns the networks that do not have the "red" tag, or do not have the "blue"
tag::

    GET /v2.0/networks?not-tags-any=red,blue

The ``tags``, ``tags-any``, ``not-tags`` and ``not-tags-any`` arguments can be
combined to build more complex queries. Example::

    GET /v2.0/networks?tags=red,blue&tags-any=green,orange

The above example returns any networks that have the "red" and "blue" tags,
 plus at least one of "green" and "orange".

Complex queries may have contradictory parameters. Example::

    GET /v2.0/networks?tags=blue&not-tags=blue

In this case we should let Neutron find these networks. Obviously
there are no such networks and Neutron will return an empty list.

CLI Examples
------------
The following examples are used to illustrate how the CLI for adding/deleting/searching tags
might look like::

   neutron tag-create --resource-type network --resource <network-id-or-name> --tag blue
   neutron tag-delete --resource-type network --resource <network-id-or-name> --tag blue
   neutron net-list --tag blue

References
==========
.. _resource tagging: http://en.wikipedia.org/wiki/Tag_(metadata)
.. _tag instances: http://specs.openstack.org/openstack/nova-specs/specs/liberty/approved/tag-instances.html
.. _guidelines: http://specs.openstack.org/openstack/api-wg/guidelines/tags.html
.. _common neutron object: https://review.openstack.org/#/c/222079/
