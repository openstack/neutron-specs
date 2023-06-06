..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================================
Flavor/service provider support for routers in the ML2/OVN L3 plugin
====================================================================

https://bugs.launchpad.net/neutron/+bug/2020823

In `Multiple L3 backends`_, functionality was added to the ML2/OVS mechanism
driver to support L3 routers implemented by different back-end drivers.
Recently, OpenStack deployers in the Telco / 5G industry have expressed a
strong interest to have the same support with the ML2/OVN mechanism driver.
This specification proposes to implement that functionality.


Problem Description
===================

Since early in the development of Neutron, the community recognized the need to
provide frameworks to enable operators and users of a cloud deployment to
configure and select from a set of options different implementations of
services such as VPNs, firewalls, routers, etc. This has led over the years to
proposals and additions to Neutron such as `Neutron/Service Type Framework`_,
`Service Flavor Framework`_ and the aforementioned `Multiple L3 backends`_.
All this is represented in the API by `Networking Flavors Framework`_,
`Service Providers`_ and the `L3 flavors extension`_.

As of the writing of this specification, operators using ML2/OVN cannot take
advantage of all these existing frameworks to offer different implementations
(flavors from here on) of vrouters to their users. As a consequence, Neutron
cannot serve a use case where Telco operators want to provide at the same time
different grades of vrouters, such as:

* OVN vrouters for their IT / back-office workloads.
* Specialized vrouters outside OVN for their 5G mobile networks.
* Specialized vrouters outside OVN for their 4G mobile networks.
* Other types of specialized vrouters.


Proposed Change
===============
The proposal is to add to the ML2/OVN L3 plugin the ability to process
different flavors of vrouters. The default flavor will be the native OVN
vrouter.

To achieve this, all the OVN specific functionality in the ML2/OVN L3 plugin
related to vrouters and floating IPs will be refactored to a separate driver.
Once this is done, the L3 plugin will be responsible for performing only the
Neutron DB processing steps related to vrouters and floating IPs, while letting
separate drivers to take care of all the backend processing. These drivers will
listen and act on events notifications sent by the refactored L3 plugin for the
creation, update and deletion of vrouters and floating IPs. Each driver will be
responsible to act only on those events that pertain to its flavor.

Cloud administrators will configure, load and associate with flavor names
drivers for backends different to OVN. The following diagram illustrates the
proposed scenario:

::

 +--------+          +------------+               +------------+    +------+    +------+    +-----------------+
 |        |          |            |     Event     |            |--->| NBDB |--->| SBDB |--->| OVN Controllers |
 |        |          |            | Notifications | Driver for |    +------+    +------+    +-----------------+
 |        |          |            |-------------->| flavor OVN |
 |        |          |            |               |            |
 |        |          |            |               +------------+
 |        |          |            |
 |        |          |            |               +------------+    +-----------+
 |        |          |            |     Event     |            |--->| Backend A |
 |  ReST  | Requests |     L3     | Notifications | Driver for |    +-----------+
 |  API   |--------->|   plugin   |-------------->|  flavor A  |
 |        |          |            |               |            |
 |        |          |            |               +------------+
 |        |          |            |
 |        |          |            |               +------------+    +-----------+
 |        |          |            |     Event     |            |--->| Backend B |
 |        |          |            | Notifications | Driver for |    +-----------+
 |        |          |            |-------------->|  flavor B  |
 |        |          |            |               |            |
 +--------+          +------------+               +------------+
                           |                             |
                           |       +-------------+       |
                           |       |             |       |
                   Updates +------>|   Neutron   |<------+ Reads
                                   |   database  |
                                   |             |
                                   +-------------+


In this diagram, backend means any combinations of agents and databases
(outside the purview of Neutron) that a driver needs to instantiate routers and
floating IPs in the data plane. This specification implementation will include
a driver for OVN and the necessary scaffolding to enable other drivers to be
configured in a Neutron deployment, but the drivers themselves and their
associated databases and agents will be provided by third parties.

Once drivers are configured, users will specify the required flavor when
creating vrouters. The OVN flavor will be the default and will be denoted by
specifying no flavor in the create request. The following commands illustrate
how to create a thrid party flavor vrouter:

::

 $ openstack network flavor list
 +--------------------------------------+----------------------------+---------+---------------+------------------------------------------------------+
 | ID                                   | Name                       | Enabled | Service Type  | Description                                          |
 +--------------------------------------+----------------------------+---------+---------------+------------------------------------------------------+
 | e47c1c5c-629b-4c48-b49a-78abe6ac7696 | user-defined-router-flavor | True    | L3_ROUTER_NAT | User defined flavor for routers in the L3 OVN plugin |
 +--------------------------------------+----------------------------+---------+---------------+------------------------------------------------------+

 $ openstack router create router-of-user-defined-flavor --flavor e47c1c5c-629b-4c48-b49a-78abe6ac7696
 +-------------------------+--------------------------------------+
 | Field                   | Value                                |
 +-------------------------+--------------------------------------+
 | admin_state_up          | UP                                   |
 | availability_zone_hints |                                      |
 | availability_zones      |                                      |
 | created_at              | 2023-06-08T20:26:01Z                 |
 | description             |                                      |
 | enable_ndp_proxy        | None                                 |
 | external_gateway_info   | null                                 |
 | flavor_id               | e47c1c5c-629b-4c48-b49a-78abe6ac7696 |
 | id                      | fe110d50-f994-4593-86a4-fc2ecca34c38 |
 | name                    | router-of-user-defined-flavor        |
 | project_id              | b807321af03f44dc808ff06bbc845804     |
 | revision_number         | 1                                    |
 | routes                  |                                      |
 | status                  | ACTIVE                               |
 | tags                    |                                      |
 | tenant_id               | b807321af03f44dc808ff06bbc845804     |
 | updated_at              | 2023-06-08T20:26:01Z                 |
 +-------------------------+--------------------------------------+

As of the writing of this specification, the openstack client doesn't allow the
specification of a flavor ID when creating a vrouter. Adding this functionality
to the client will be part of this specification's implementation.

The OVN flavor driver will be loaded by default when the L3 plugin starts.
This is an example of the steps a cloud administrator will follow to configure
and load drivers for other flavors:

#. Add the service provider to neutron.conf::

     [service_providers]
     service_provider = L3_ROUTER_NAT:user-defined:neutron.services.ovn_l3.service_providers.user_defined.UserDefined

#. Re-start the neutron server and verify the user defined provider has been
   loaded::

     $ openstack network service provider list
     +---------------+--------------+---------+
     | Service Type  | Name         | Default |
     +---------------+--------------+---------+
     | L3_ROUTER_NAT | user-defined | False   |
     | L3_ROUTER_NAT | ovn          | True    |
     +---------------+--------------+---------+

#. Create a service profile for the router flavor::

     $ openstack network flavor profile create --description "User defined router flavor profile" --enable --driver neutron.services.ovn_l3.service_providers.user_defined.UserDefined
     +-------------+--------------------------------------------------------------------+
     | Field       | Value                                                              |
     +-------------+--------------------------------------------------------------------+
     | description | User defined router flavor profile                                 |
     | driver      | neutron.services.ovn_l3.service_providers.user_defined.UserDefined |
     | enabled     | True                                                               |
     | id          | a717c92c-63f7-47e8-9efb-6ad0d61c4875                               |
     | meta_info   |                                                                    |
     | project_id  | None                                                               |
     +-------------+--------------------------------------------------------------------+

#. Create the router flavor::

     $ openstack network flavor create --service-type L3_ROUTER_NAT --description "User defined flavor for routers in the L3 OVN plugin" user-defined-router-flavor
     +---------------------+------------------------------------------------------+
     | Field               | Value                                                |
     +---------------------+------------------------------------------------------+
     | description         | User defined flavor for routers in the L3 OVN plugin |
     | enabled             | True                                                 |
     | id                  | e47c1c5c-629b-4c48-b49a-78abe6ac7696                 |
     | name                | user-defined-router-flavor                           |
     | service_profile_ids | []                                                   |
     | service_type        | L3_ROUTER_NAT                                        |
     +---------------------+------------------------------------------------------+

#. Add service profile to router flavor::

     $ openstack network flavor add profile user-defined-router-flavor a717c92c-63f7-47e8-9efb-6ad0d61c4875

REST API impact
---------------

No REST API impact is expected.

DB Impact
---------

No DB impact is expected.

Implementation
==============

Assignee(s)
-----------

* Miguel Lavalle irc: mlavalle, email: mlavalle@redhat.com

Work Items
----------

* Re-factor the OVN vrouters and floating IPs (NAT) functionality embedded in
  the L3 plugin to a separate driver.
* Implement a drivers controller for the L3 plugin that will load automatically
  the driver for OVN routers and floating IPs.
* Update the the ML2/OVN maintenance task so vrouters of third party flavors
  are not synched with the NBDB.
* Add functionality to the openstack client to enable users to specify a flavor
  when creating a vrouter.
* Implement tests.
* Write documentation.


References
==========

Provided inline in the text above.

.. _Multiple L3 backends: https://review.opendev.org/q/topic:bp%252Fmulti-l3-backends
.. _Neutron/Service Type Framework: https://wiki.openstack.org/wiki/Neutron/ServiceTypeFramework
.. _Service Flavor Framework: https://specs.openstack.org/openstack/neutron-specs/specs/liberty/neutron-flavor-framework.html
.. _Networking Flavors Framework: https://docs.openstack.org/api-ref/network/v2/index.html#networking-flavors-framework-v2-0-current-flavor-service-profile
.. _Service Providers: https://docs.openstack.org/api-ref/network/v2/index.html#service-providers
.. _L3 flavors extension: https://docs.openstack.org/api-ref/network/v2/index.html#l3-flavors-extension-l3-flavors
