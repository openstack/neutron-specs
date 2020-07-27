..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Port NUMA affinity policy
=========================

https://bugs.launchpad.net/neutron/+bug/1886798

Add a new parameter to "port" object setting the NUMA affinity policy.


Problem Description
===================

Currently Nova allows to define the "numa_affinity_policy" for PCI devices.
Those PCI devices could be, for example, network cards. When a port is created
using this physical device (an SR-IOV port with VNIC type "direct",
"direct-physical", "macvtap" or "virtio-forwarder"), Nova will select the host
to spawn the virtual machine depending on the host NUMA topology availability
and the NUMA nodes associated to this PCI network device [1]_.

However this filtering cannot be done currently for other backends than SR-IOV.
For example, Open vSwitch with DPDK will run the PMD threads attached to
specific CPU cores (and to specific NUMA nodes) [2]_. Nova represents
this information assigning a set of NUMA nodes to each defined physical or
tunneled network [3]_. If the port NUMA affinity policy is provided, Nova will
enforce it during the scheduling.


Proposed Change
===============

The goal of this spec is to create a "numa_affinity_policy" parameter
applicable to all ports. This information will be provided to Nova that will
use it while scheluding the virtual machine, independently of the network
backend.

Of course, Nova should have the network backend NUMA information topology. For
example, since [3]_, Nova can be statically configured with the physical
networks NUMA location. With the port "numa_affinity_policy" parameter, the
Nova scheduler will filter those hosts matching the required policy.

This spec covers the missing cases defined in [4]_. Defining the NUMA
affinity policy per port (interface), includes all type of ports, not only the
SR-IOV interfaces. It also avoids defining the NUMA affinity in the Nova
flavor; instead of this, a generic flavor can be used with ports with
different policies.

This spec proposes to add a new parameter to the "port" object. This parameter,
"numa_affinity_policy", will be an string defining the NUMA affinity policy,
based on the PCINUMAAffinityPolicy [5]_ enum defined in Nova. That field could
have different values: "required", "legacy" and "preferred".

This information will be populated in the "port" dictionary when informing to
Nova. This parameter will be a trait added in the "resource_request" parameter,
in the "required" list. This parameter was included in the Nova microversion
2.72 [6]_. This information will be used during the server scheduling to
filter the host requirements using the Placement API.

By default, the value of this new parameter will be "None", to keep
backwards compatibility. In this case, no trait will be added to the
"resource_request" parameter.

.. code-block:: python

    port_resource['resource_request'] = {
        'required': [os_traits.COMPUTE_NUMA_POLICY_REQUIRED]}


Data Model Impact
-----------------

A new table, "portnumaaffinitypolicy", will be created:

==================== ======== ==== ============= =====================
Attribute Name       Type     CRUD Default Value Validation/Conversion
==================== ======== ==== ============= =====================
port_id              uuid-str R
numa_affinity_policy str      CRU  None          enum (including None)
==================== ======== ==== ============= =====================

This child table will depend on the "ports" table. Each row will have a 1:1
relationship with a "port" row and will be deleted when the "port"
row is deleted too.


REST API Impact
---------------

The parameter "numa_affinity_policy" in the "port" API:

.. code-block:: python

    NUMA_AFFINITY_POLICY_VALUES = (None, 'required', 'preferred', 'legacy')

    RESOURCE_ATTRIBUTE_MAP = {
        'port': {
            'numa_affinity_policy': {
                'allow_post': True,
                'allow_put': True,
                'validate': {'type:values': NUMA_AFFINITY_POLICY_VALUES}
                'default': None,
                'is_visible': True}
        }
    }


The parameter can be updated only if the port is not bound. That check does
not depend on the API but on the server.


Security Impact
---------------

None


Performance Impact
------------------

None


Operators CLI Impact
--------------------

An additional parameter will be added to the OSC "port" CLI interface, in the
create, set and unset commands.

For logging resource::

    openstack port create [--numa-policy-required | --numa-policy-preferred |
                           --numa-policy-legacy]

    openstack port set [--numa-policy-required | --numa-policy-preferred |
                        --numa-policy-legacy]

    openstack port unset --numa-policy


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Rodolfo Alonso Hernandez <ralonsoh@redhat.com> (IRC: ralonsoh)


Testing
=======

* Unit Test
* Functional test
* API test


Documentation Impact
====================

User Documentation
------------------

* Add CLI usage into the networking guide for operator.


References
==========

.. [1] https://specs.openstack.org/openstack/nova-specs/specs/queens/implemented/share-pci-between-numa-nodes.html
.. [2] http://docs.openvswitch.org/en/latest/intro/install/dpdk/
.. [3] https://specs.openstack.org/openstack/nova-specs/specs/rocky/implemented/numa-aware-vswitches.html
.. [4] https://specs.openstack.org/openstack/nova-specs/specs/ussuri/implemented/vm-scoped-sriov-numa-affinity.html#alternatives
.. [5] https://github.com/openstack/nova/blob/d4c857dfcb1ccfa5410de55671e69c722bbc990e/nova/objects/fields.py#L740-L746
.. [6] https://docs.openstack.org/api-guide/compute/port_with_resource_request.html
