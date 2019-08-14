..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

Improve Extraroute API
======================

https://bugs.launchpad.net/neutron/+bug/1826396

As discussed in an `openstack-discuss thread
<http://lists.openstack.org/pipermail/openstack-discuss/2019-April/005121.html>`_
we could improve the extraroute API to better support Neutron API clients,
especially Heat.

Problem Description
-------------------

The first problem is that the current extraroute API does
not allow atomic additions/deletions of particular routing
table entries. In the current API the routes attribute of a
router (containing all routing table entries) `must be updated at once
<https://developer.openstack.org/api-ref/network/v2/?expanded=update-router-detail#update-router>`_.
Therefore additions and deletions must be performed on the client
side. Therefore multiple clients race to update the routes attribute
and updates may get lost.

This problem can be easily (but nondeterministically) reproduced by a
`simple HOT template as included in the References section below
<#extraroute-concurrent-yaml>`_.

::

  $ openstack stack create --wait -t extraroute-concurrent.yaml stack0
  $ openstack stack resource list stack0 --filter name=router0 -f value -c physical_resource_id \
    | xargs -r openstack router show -f value -c routes

The second problem perceived by programmatic users of the Neutron API
is that the ownership of extra routes cannot be expressed because they
don't have unique identifiers. For example consider two Heat stacks
(or any other user of Neutron's API) needing and therefore creating
the same extra route. When one of them is deleted the extra route gets
deleted for both unless we have a way to track multiple needs for the
same extra route. Since addressing the second problem would involve
changing the conceptual abstraction of an extra route this problem is
not addressed in this specification.

Proposed Change
---------------

This RFE proposes to expose the extraroute functionality of Neutron in a
new way beyond the current 'routes' attribute of routers.

Introduce a new API extension: extraroutes-atomic

::

  PUT /v2.0/routers/{router_id}/add_extraroutes

  { "router":
    { "routes":
      [ { "destination": "179.24.1.0/24",
          "nexthop": "172.24.3.99" },
        ...
      ]
    }
  }

::

  200 OK

  { "router":
    { "id": "1ecae6b8-be64-11e9-98ba-733d5460217b",
      "name": "router1",
      "routes":
      [ { "destination": "179.24.1.0/24",
          "nexthop": "172.24.3.99" },
        ...
      ],
      ...
    }
  }

::

  PUT /v2.0/routers/{router_id}/remove_extraroutes

  { "router":
    { "routes":
      [ { "destination": "179.24.1.0/24",
          "nexthop": "172.24.3.99" },
        ...
      ]
    }
  }

::

  200 OK

  { "router":
    { "id": "1ecae6b8-be64-11e9-98ba-733d5460217b",
      "name": "router1",
      "routes":
      [ remaining routes ],
      ...
    }
  }

Partial failures are not allowed. If the addition or removal of any routing
table entry fails then the whole update is reverted.

If a routing table entry to be added (removed) is already present (missing)
that is not an error, but considered a successful update.

Handling of duplicate and overlapping routes is not changed from the
current behavior. That is exact duplicates are rejected, but overlapping
routes are allowed.

Deleting a router cascades to deleting all extra routes of that router.

The DB and OVO layer in Neutron is ready for this change, the routes
attribute is is already de-composed to its own table:

::

  mysql> describe routerroutes;
  +-------------+-------------+------+-----+---------+-------+
  | Field       | Type        | Null | Key | Default | Extra |
  +-------------+-------------+------+-----+---------+-------+
  | destination | varchar(64) | NO   | PRI | NULL    |       |
  | nexthop     | varchar(64) | NO   | PRI | NULL    |       |
  | router_id   | varchar(36) | NO   | PRI | NULL    |       |
  +-------------+-------------+------+-----+---------+-------+

Further changes included:

* Documentation: api-ref.
* Unit tests.
* Tempest test in neutron-tempest-plugin.
* Adapt python-neutronclient.
* Adapt openstackclient.
* Adapt openstacksdk.
* I also hope to improve Heat's OS::Neutron::Extraroute resource building
  on this work, but that will be described in its own Heat blueprint.

Deprecations
~~~~~~~~~~~~

As soon as the new API is merged direct updates to the 'routes' attribute
of a router are deprecated, but keeping in line with the long standing
Neutron tradition of not making backward incompatible API changes the
old extraroutes extension is *not* removed.

Other Impact
~~~~~~~~~~~~

No impact expected on upgrades.
No impact expected on configuration.
No impact expected on RPC.

Alternatives
~~~~~~~~~~~~

Compare and Swap
++++++++++++++++

Neutron has `compare-and-swap API update logic
<https://bugs.launchpad.net/neutron/+bug/1703234>`_. However using that
to solve this problem is awkward on the client side (retry until success
if the routes attibute change while the client edited it). Plus as the
number of racing clients grow use of the compare-and-swap API is going
to generate unnecessary load on API as update requests must be thrown
away and retried.

Additionally current client code bases do not have support built in for
compare-and-swap operations.

For other alternatives please see the review discussion of this specification.

References
----------

* RFE bug report of this spec: https://bugs.launchpad.net/neutron/+bug/1826396
* Heat story to consume the API proposed here: https://storyboard.openstack.org/#!/story/2005522

extraroute-concurrent.yaml
~~~~~~~~~~~~~~~~~~~~~~~~~~

::

  description: test of extraroute concurrency
  heat_template_version: 2015-04-30

  resources:

    net0:
      type: OS::Neutron::Net

    subnet0:
      type: OS::Neutron::Subnet
      properties:
        network: { get_resource: net0 }
        cidr: 10.0.0.0/24

    router0:
      type: OS::Neutron::Router

    routerinterface0:
      type: OS::Neutron::RouterInterface
      properties:
        router: { get_resource: router0 }
        subnet: { get_resource: subnet0 }

    extraroute0:
      type: OS::Neutron::ExtraRoute
      properties:
        destination: 10.1.0.0/24
        nexthop: 10.0.0.10
        router_id: { get_resource: router0 }

    extraroute1:
      type: OS::Neutron::ExtraRoute
      properties:
        destination: 10.1.1.0/24
        nexthop: 10.0.0.11
        router_id: { get_resource: router0 }

    extraroute2:
      type: OS::Neutron::ExtraRoute
      properties:
        destination: 10.1.2.0/24
        nexthop: 10.0.0.12
        router_id: { get_resource: router0 }

    extraroute3:
      type: OS::Neutron::ExtraRoute
      properties:
        destination: 10.1.3.0/24
        nexthop: 10.0.0.13
        router_id: { get_resource: router0 }

    extraroute4:
      type: OS::Neutron::ExtraRoute
      properties:
        destination: 10.1.4.0/24
        nexthop: 10.0.0.14
        router_id: { get_resource: router0 }

    extraroute5:
      type: OS::Neutron::ExtraRoute
      properties:
        destination: 10.1.5.0/24
        nexthop: 10.0.0.15
        router_id: { get_resource: router0 }

    extraroute6:
      type: OS::Neutron::ExtraRoute
      properties:
        destination: 10.1.6.0/24
        nexthop: 10.0.0.16
        router_id: { get_resource: router0 }

    extraroute7:
      type: OS::Neutron::ExtraRoute
      properties:
        destination: 10.1.7.0/24
        nexthop: 10.0.0.17
        router_id: { get_resource: router0 }

    extraroute8:
      type: OS::Neutron::ExtraRoute
      properties:
        destination: 10.1.8.0/24
        nexthop: 10.0.0.18
        router_id: { get_resource: router0 }

    extraroute9:
      type: OS::Neutron::ExtraRoute
      properties:
        destination: 10.1.9.0/24
        nexthop: 10.0.0.19
        router_id: { get_resource: router0 }
