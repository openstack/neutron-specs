==================================
ML2 Provider-network partial specs
==================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/provider-network-partial-specs

This blueprint will allow creation of provider networks with provider
attributes partially provided. This capability will be implemented only for
the ML2 plugin.



Problem description
===================

Currently, all provider attributes for tenant networks are chosen by neutron,
while for provider networks **all** the provider attributes must be specified
by admins. For provider networks, admins cannot specify some provider
attributes and delegate the choice of remaining ones to neutron-server. This
means that admins must know the valid and unallocated physical_networks and
segmentation_ids when creating provider networks.

This blueprint will let admins specify some provider attributes, and let
neutron choose the remaining attributes from pools of allowable values.

Use cases:

* Create a (vlan) network for baremetal deployment without focusing
  on physical_network/segmentation_id choices,
* Create "service" networks (typically networks for storage) which
  are provided over vlan on a specific physical_network without focusing on
  segmentation_id choice,
* Create gre/vxlan networks for testing.


Proposed change
===============

Vlan/gre/vxlan ML2 type drivers will be extended in order to support partially
providing provider/multi-provider network attributes on network create, and
let type drivers choose a network from tenant network pools.

More precisely type drivers will choose a network from tenant network pools
matching provided attributes and allocate it. If no candidate is found the type
driver will raise a NoNetworkAvailable exception. The network delete process
will not change: the network will be returned to its tenant network pool.


Alternatives
------------

An alternative solution would be to provide alternative type driver
implementations for previous network types allowing to partially specify
provider attributes. This solution does not require new configuration options
but ML2 type manager must check that at most one type driver is provided per
network type.


Data model impact
-----------------

None

REST API impact
---------------

This blueprint reduces constraints on provider network attributes on network
create:

* If provider:network_type=vlan is specified, then provider:physical_network
  and provider:segmentation_id are optional::

    neutron net-create net-vlan --provider:network_type=vlan
    neutron net-create net-storage --provider:network_type=vlan\
        --provider:physical_network=storage

* If provider:network_type=gre is specified, then provider:segmentation_id is
  optional::

    neutron net-create net-gre --provider:network_type=gre

* If provider:network_type=vxlan is specified, then provider:segmentation_id is
  optional::

    neutron net-create net-vxlan --provider:network_type=vxlan

The same constraint reduction applies on multi-provider attributes on network
create::

    neutron net-create net-multi --segments type=dict list=true\
        provider:network_type=vlan,provider:physical_network=multi\
        provider:network_type=gre\


Security impact
---------------

None

Provider/multi-provider network default policy will not change.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
Cedric Brandily <cbrandily>

Work Items
----------

1. Update ML2 plugin/type manager in order to allow reserve_provider_network
   to return allocated network
2. Extend vlan type driver to support partial specs
3. Extend gre/vxlan type driver to support partial specs
4. Update ML2 multi-provider to support partial specs


Dependencies
============

None


Testing
=======

The code will be covered by unit tests.

Tempest tests will be provided. Tempest must be aware of network types
supported by tested deployment in order to tests partial specs on all
supported network types. The new tempest option **partial_specs_scenario**
will be define to configure supported network types:

* Disable partial specs tests (default value)::

    partial_specs_scenario =

* Enable vlan provider networks partial specs tests::

    partial_specs_scenario = vlan

* Enable vlan and gre provider networks partial specs tests::

    partial_specs_scenario = vlan,gre


Documentation Impact
====================

Document deployer impacts.


References
==========

https://review.openstack.org/#/q/topic:bp/provider-network-partial-specs,n,z
