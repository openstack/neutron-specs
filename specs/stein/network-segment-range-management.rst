..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Network Segment Range Management
================================

Launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/network-segment-range-management

Currently, network segment ranges are configured as an entry in ML2 config
file [1]_ that is statically defined for tenant network allocation and
therefore must be managed as part of the host deployment and management. When a
normal tenant user creates a network, Neutron assigns the next free
segmentation ID (VLAN ID, VNI etc.) from the configured segment ranges. Only an
administrator can assign a specific segment ID via the provider extension.

This spec introduces an extension which exposes the segment range management to
be administered via the Neutron API. In addition, it introduces the ability for
the administrator to control the segment ranges globally or on a per-tenant
basis.


Problem Description
===================

Self-service networks, aka tenant networks primarily enable general
(non-privileged) projects to manage networks without involving administrators.
These networks are entirely virtual and require virtual routers to interact
with provider and external networks such as the Internet. In most cases,
self-service networks use overlay protocols such as VXLAN or GRE because they
can support many more networks than layer-2 segmentation using VLAN tagging
(802.1q) which typically requires additional configuration of physical network
infrastructure. Nevertheless, in some other cases, OpenStack Neutron also
supports the self-service network isolation using VLAN etc. [2]_.

The current Neutron implementation sets entries in a config file to statically
define network segment ranges for tenant network allocation. It does not
support another dynamic way (e.g. at the API level) after initialization.
Neutron services have to be restarted in order to make any change to the
network segment ranges take effect. However, cloud network infrastructure
deployments can be complex and non-homogeneous. This rigidity brings about
complexity and difficulty to cloud administrators when dealing with varied
requirements.

Network segment range management provides an additional level of flexibility at
the API level which enables full network orchestration of all the types of
tenant networks using standard REST API rather than needing to interact with
the host configuration directly. Apart from this, for VLAN tenant networks in
particular, network segment range management enables a control over the
management of the cloud/network infrastructure for tenant networks. It also
facilitates per-tenant assignments (including shared). This enables a cloud
administrator to directly control the VLAN tenant segment mapping, and
ultimately the underlying L2/L3 path of the tenant traffic without exposing
specific network segment range information to the tenant user. The segment
allocations for tenant networks will refer to the network segment ranges which
are assigned to that tenant by an administrator.

Several sample use cases are demonstrated below.

Use Case I
----------

Enable cloud administrators to create and assign per-tenant network segment
ranges. This gives an administrator the privilege to manage the underlying
network segment ranges. When a tenant creates a network, it will allocate a
segment ID from the segment ranges assigned to the tenant or shared if no
tenant specific range available. It helps map the VMs created by that tenant
with an explicit set of existing networks, for privacy or dedicated business
connection needs. One example illustration is shown below:

::

                             +------------------+
                             | Physical Network |
      +------------+---------+ Infrastructures  +-------------------+
      |            |         +---------+--------+                   |
      |            |                   |                            |
      |            |                   |                            |
 +----+----+  +----+----+         +----+----+                 +----+----+
 | Tenant 0|  | Tenant 1|.........| Tenant k|.................| Tenant n|
 +----+----+  +----+----+         +----+----+                 +----+----+
      |            |                   |                           |
      |            |                   |                           |
      + range-0    + range-1           + range-k1                  + range-n
                                         range-k2

* One cloud is connecting and mapping with a large number of existing l2
  physical network infrastructures.

* n+1 tenants: Tenant 0, ...Tenant k..., Tenant n are available in this cloud.

* Each tenant needs to be assigned with one (or more) network segment range(s)
  respectively (range-0, ...range-k1..., ...range-k2..., range-n) by the cloud
  administrator, so that the networks it creates can reach the corresponding
  dedicated physical network infrastructure it would like to connect to
  according to different business requirements.

A possible workflow is presented as follows:

1. Cloud administrator lists all network segment ranges existing::

    openstack network segment range list

2. If no network segment range created for one target tenant, then cloud
   administrator could create one for it::

    openstack network segment range create
                --name <range_name>
                --shared <shared>
                --project <project_id>
                --network_type <network_type>
                --physical_network <physical_network_name>
                --minimum <range_minimum>
                --maximum <range_maximum>

3. Now a normal tenant user can create networks in regular way [3]_. The
   network created will automatically allocate a segment ID from the segment
   ranges assigned to the tenant (per step 3) or shared if no tenant specific
   range available.

This helps deciding the correct VLAN mapping when multiple tenants exist. It
also provides the possibility to dynamically create a network segment range for
each tenant, even if it's not pre-deployed in the host configuration.

Use Case II
-----------

In some VLAN tenant network scenarios, it is not uncommon that the existing
physical network infrastructures’ segment configurations have been changed.
Thus, it is a must for cloud administrators to update the VLAN allocation range
for tenant networks at the same time in order to keep aligned with the
connection or mapping. Hence offering the ability to dynamically manage segment
ranges of self-service networks, as this spec proposes, is imperative for VLAN
tenant networks and is definitely a plus for other overlay tenant networks.

A possible workflow is presented as follows:

1. Cloud administrator lists all network segment ranges existing and
   identifies the one needs to update::

    openstack network segment range list

2. Cloud administrator updates a network segment range based on the actual
   requirement changing::

    openstack network segment range set
                --name <range_name>
                --minimum <range_minimum>
                --maximum <range_maximum>
                <network_segment_range-id>

3. Now a normal tenant user can create networks in regular way [3]_. The network
   created will automatically allocate free segment ID from the updated network
   segment ranges available.


Proposed Change
===============

To address the above use cases, this spec introduces a new resource called
network_segment_ranges together with its implementation.

Currently by default, all pre-configured segment information (e.g.
network_vlan_ranges, vni_ranges etc. defined in [1]_) is loaded into
"ml2_xxx_allocations" DB tables by ML2 type drivers once the Neutron services
are up. For all self-service networks’ segmentation ID sync, allocations and
releases, they will be based on this information.

All this information will be maintained. The network segment ranges introduced
in this spec will augment this initial allocation that is loaded from the
configuration file to become part of the shared ranges. It proposes an API way
that user with administrative privileges can create and manage various network
segment ranges for all network types supported by ML2.

Data Model Impact
-----------------

The following new table is added as part of the network network management
feature::

    CREATE TABLE network_segment_ranges (
      id CHAR(36) NOT NULL PRI KEY,
      name VARCHAR(255),
      default BOOL NOT NULL,
      shared BOOL NOT NULL,
      project_id VARCHAR(255) NOT NULL,
      network_type ENUM('vlan', 'vxlan', 'gre', 'geneve') NOT NULL,
      physical_network VARCHAR(64),
      minimum INT,
      maximum INT
    );

For different network types, the validation strategies and responses should
have the following variants:

* VLAN: minimum = 1, maximum = 4094.

* VXLAN: minimum = 1, maximum = 2 ** 24 - 1.

* GRE: minimum = 1, maximum = 2 ** 32 - 1.

* Geneve: minimum = 1, maximum = 2 ** 24 - 1.

Notes: Other validation rules like minimum <= maximum are always applicable and
should be paid attention to. Most of the cited above have been supported in
neutron_lib.plugins.utils.

Mixin classes to add the network segment range management extension should be
provided. The DB operation logic should be handled by the ML2 type manager and
the type drivers. For the values present in the existing ML2 configuration
options [1]_ (e.g. ml2_type_vlan, ml2_type_vxlan etc.), they will be loaded as
`shared` and `default` segment ranges into network_segment_ranges DB in order
to provide backward compatibility for initial deployment when this extension is
present.

Resource Extension
------------------
The following new resource is being introduced and its attributes maps would be
like:

.. code-block:: python

  NETWORK_TYPE_LIST = [TYPE_VLAN, TYPE_VXLAN, TYPE_GRE, TYPE_GENEVE]
  RESOURCE_ATTRIBUTE_MAPS = {
    'network_segment_ranges': {
      'id': {'allow_post': False, 'allow_put': False,
             'validate': {'type:uuid': None},
             'is_visible': True,
             'is_filter': True,
             'is_sort_key': True,
             'primary_key': True},
      'name': {'allow_post': True, 'allow_put': True,
               'validate': {
                 'type:string': db_const.NAME_FIELD_SIZE},
               'default': '', 'is_visible': True, 'is_filter': True,
               'is_sort_key': True},
      'default': {'allow_post': False, 'allow_put': False,
                  'convert_to': converters.convert_to_boolean,
                  'default': False,
                  'is_visible': True},
      'shared': {'allow_post': True, 'allow_put': False,
                 'convert_to': converters.convert_to_boolean,
                 'is_visible': True, 'default': True},
      'project_id': {'allow_post': True, 'allow_put': False,
                     'validate': {
                     'type:string': db_const.PROJECT_ID_FIELD_SIZE},
                     'required_by_policy': True,
                     'is_filter': True,
                     'is_sort_key': True,
                     'is_visible': True},
      'network_type': {'allow_post': True, 'allow_put': False,
                       'validate': {'type:values': NETWORK_TYPE_LIST},
                       'default': constants.ATTR_NOT_SPECIFIED,
                       'is_filter': True,
                       'is_visible': True},
      'physical_network': {'allow_post': True, 'allow_put': False,
                           'validate': {
                             'type:string': PHYSICAL_NETWORK_MAX_LEN},
                           'default': constants.ATTR_NOT_SPECIFIED,
                           'is_filter': True,
                           'is_visible': True},
      'minimum': {'allow_post': True, 'allow_put': True,
                  'convert_to': converters.convert_to_int, 'is_visible': True},
      'maximum': {'allow_post': True, 'allow_put': True,
                  'convert_to': converters.convert_to_int, 'is_visible': True},
      'used': {'allow_post': False, 'allow_put': False,
               'is_visible': True},
      'available': {'allow_post': False, 'allow_put': False,
                    'convert_to': attr.convert_none_to_empty_list,
                    'is_visible': True},
    },
  }

REST API Impact
---------------

The idea is to add a new resource extension with the below defined attributes.
Resource extension network_segment_ranges:

+-----------------+--------+-----+--------+-----------+-----------------------+
|  Attribute Name |  Type  | Req |  CRUD  | Default   |     Description       |
|                 |        |     |        | Value     |                       |
+=================+========+=====+========+===========+=======================+
| id              | String | N/A |  R     | Generated | Identifier of network |
|                 |        |     |        |           | segment range         |
+-----------------+--------+-----+--------+-----------+-----------------------+
| name            | String | No  |  CRU   | ''        | Name of network       |
|                 |        |     |        |           | segment range         |
+-----------------+--------+-----+--------+-----------+-----------------------+
| default         | Bool   | No  |  R     | False     | Default network       |
|                 |        |     |        |           | segment range that is |
|                 |        |     |        |           | loaded from the host  |
|                 |        |     |        |           | ML2 config file [1]_  |
+-----------------+--------+-----+--------+-----------+-----------------------+
| shared          | Bool   | Yes |  CR    | True      | Shared with other     |
|                 |        |     |        |           | projects              |
+-----------------+--------+-----+--------+-----------+-----------------------+
| project_id      | String | No  |  CR    | Current   | Owner of network      |
|                 |        |     |        | project_id| range. Optional when  |
|                 |        |     |        |           | `shared` is True.     |
+-----------------+--------+-----+--------+-----------+-----------------------+
| network_type    | Enum   | Yes |  CR    | None      | VLAN, VxLAN, GRE      |
|                 |        |     |        |           | Geneve                |
+-----------------+--------+-----+--------+-----------+-----------------------+
| physical_network| String | No  |  CR    | None      | Optional. Only        |
|                 |        |     |        |           | applicable for VLAN.  |
+-----------------+--------+-----+--------+-----------+-----------------------+
| minimum         | INT    | Yes |  CRU   | None      | Floor integer of the  |
|                 |        |     |        |           | segment range         |
+-----------------+--------+-----+--------+-----------+-----------------------+
| maximum         | INT    | Yes |  CRU   | None      | Ceiling integer of the|
|                 |        |     |        |           | segment range         |
+-----------------+--------+-----+--------+-----------+-----------------------+
| used            | Dict   | No  |  R     | {}        | Mapping of which      |
|                 |        |     |        |           | segmentation ID in the|
|                 |        |     |        |           | range is used by which|
|                 |        |     |        |           | tenant                |
+-----------------+--------+-----+--------+-----------+-----------------------+
| available       | List   | No  |  R     | []        | List of available     |
|                 |        |     |        |           | segmentation IDs in   |
|                 |        |     |        |           | this segment range    |
+-----------------+--------+-----+--------+-----------+-----------------------+

To specify a range with single item, min equals to max can do the trick. For
discrete segment ranges of one given network type, they are represented as
several ones, each with a min and a max.

The following network segment range management Rest APIs will be provided in
line with the new resources previously introduced:

* List all network segment ranges.
  GET /v2.0/network_segment_ranges

::

    GET /v2.0/network_segment_ranges
    Accept: application/json
    {
      "network_segment_ranges": [
        {
          "id": "d23abc8d-2991-4a55-ba98-2aaea84cc72f",
          "name": "network_segment_range_physnet1",
          "default": False,
          "shared": False,
          "project_id": "45977fa2dbd7482098dd68d0d8970117",
          "network_type": "vlan",
          "physical_network": "physnet1",
          "minimum": 100,
          "maximum": 105,
          "used": {"100": "07ac1127ee9647d48ce2626867104a13",
                   "101": "d4fa62aa47d340d98d076801aa7e6ec4"},
          "available": [102, 103, 104, 105],
        }
      ]
    }

* List a network segment range information.
  GET /v2.0/network_segment_ranges/<network_segment_range-id>

::

    GET /v2.0/network_segment_ranges/d23abc8d-2991-4a55-ba98-2aaea84cc72f
    Accept: application/json
    {
      "network_segment_range": {
        "id": "d23abc8d-2991-4a55-ba98-2aaea84cc72f",
        "name": "network_segment_range_physnet1",
        "default": False,
        "shared": False,
        "project_id": "45977fa2dbd7482098dd68d0d8970117",
        "network_type": "vlan",
        "physical_network": "physnet1",
        "minimum": 100,
        "maximum": 105,
        "used": {"100": "07ac1127ee9647d48ce2626867104a13",
                 "101": "d4fa62aa47d340d98d076801aa7e6ec4"},
        "available": [102, 103, 104, 105],
      }
    }

* Create a network segment range for a given tenant.
  POST /v2.0/network_segment_ranges

::

    POST /v2.0/network_segment_ranges
    Accept: application/json
    {
      "network_segment_range": {
        "name": "network_segment_range_physnet1",
        "shared": False,
        "project_id": "45977fa2dbd7482098dd68d0d8970117",
        "network_type": "vlan",
        "physical_network": "physnet1",
        "minimum": 100,
        "maximum": 200,
      }
    }

* Delete a network segment range by id.
  DELETE /v2.0/network_segment_ranges/<network_segment_range-id>

  Normal Response Code: 204

  Error Response Codes: Unauthorized (401), Not Found (404), Conflict (409).
  The Conflict error response is returned when an operation is performed while
  the network segment range has resource allocated within it.

  This operation does not require a request body.

  This operation does not return a response body.

::

    DELETE /v2.0/network_segment_ranges/d23abc8d-2991-4a55-ba98-2aaea84cc72f
    Accept: application/json

* Update a network segment range with given data.
  PUT /v2.0/network_segment_ranges/<network_segment_range-id>

::

    PUT /v2.0/network_segment_ranges/d23abc8d-2991-4a55-ba98-2aaea84cc72f
    Accept: application/json
    {
      "network_segment_range": {
        "minimum": 200,
        "maximum": 300,
      }
    }

Command Line Client Impact
--------------------------

Openstack Client would add network segment range management related CLIs. They
should be admin only CLI commands. For example::

    openstack network segment range list
                [--used | --not-used]
                [--available | --not-available]

    openstack network segment range show <network_segment_range-id>

    openstack network segment range create
                --shared <shared>
                --network_type <network_type>
                --minimum <range_minimum>
                --maximum <range_maximum>
                [--name <range_name>]
                [--project_id <project_id>]
                [--physical_network <physical_network>]

    Notes: Arguments --name, --project_id are optional, the project_id can be
           taken from the context if not given; --physical_network is optional
           and only applicable for VLAN. All the other parameters should be
           required.

    openstack network segment range set
                [--name <range_name>]
                [--minimum <range_minimum>]
                [--maximum <range_maximum>]
                <network_segment_range-id>

    openstack network segment range delete <network_segment_range-id>

Other Impact
------------

* ML2 plugin, plugin manager and type drivers will need to be refined and added
  with several new methods correspondingly in order to support this feature.

* When this extension is loaded, the Neutron server will populate the proposed
  network_segment_ranges DB table with the ranges defined within the existing
  ML2 configuration as `default` and `shared` ranges. These `default` ranges
  are read-only and an administrator can make per-tenant segment range
  assignment based on this information. When a Neutron server starts/restarts,
  the `default` segment ranges will be reloaded and be visible to all servers.
  Interactions from the REST APIs will always operate based on the segment
  ranges defined within the database.

* Validation work is needed for quite a few cases, including but not limited
  to:

  * Admin privilege should always be checked before performing any network
    segment range operation cited previously.

  * When deleting one network segment range, the operation should be rejected
    if one of the network segments is in use.

  * It is allowed to update one network segment range if it does not impact the
    in-use segment allocated within the range (e.g. enlarging that range).
    Otherwise, we should fail the updating.

  * We're maintaining the consistency by "ml2_xxx_allocations" DB tables and
    relying on them to do the eventual validation and the underlying segment
    allocation. This means network segment ranges can be configured before
    agents are actually mapped to specific physical network mappings.

* Theoretically, multiple network segment ranges can be created for one
  tenant (while one network segment range cannot be owned by several tenants).
  If a tenant has more than one segment range configured, it would pick up the
  next free segmentation ID (VLAN ID, VNI etc.) from all its owned network
  segment ranges.

  Backwards compatibility comes from having the default behavior of segment
  ranges being assigned as a `shared` resource to tenants. If both of `shared`
  and `specified` segment range resources are exposed to a tenant, the
  `specified` should override the `shared`.

* The extension will be optional (an option in [4]_ with functionality disabled
  by default should be added), in which case the existing ML2 configuration
  options will be applicable.

Other End User Impact
---------------------

Users with admin privileges will be able to dynamically manage network segment
ranges for which supports segmentation, all tenants and tenant networks. If no
dynamic network segment range is created for a given tenant, or the feature is
disabled due to the backwards compatibility consideration, there will be no
impact to end users.

Other Deployer Impact
---------------------

Deployers should have an option available to enable or disable this
functionality so that they can continue to use the configuration file as
before. They also need to be strongly warned to update their operational
documentation to ensure that the new network segment information is managed
using the correct facility. If the feature is disabled, nothing at the
deployment level would be impacted.

Performance Impact
------------------

Performance testing must be conducted to see what is the overhead of enabling
this feature, of course if the feature is disabled no performance impact should
be noticed.


Implementation
==============

Assignee(s)
-----------

* Kailun Qin <kailun.qin@intel.com>

Work Items
----------

* Adjust the DB model and add the new table.
* Extend current API.
* Modify type drivers and all related references.
* Add related tests.
* Add CLI openstackclient.
* Documentation.


Dependencies
============

None


Testing
=======

Unit tests, functional tests, API tests and scenario tests are necessary.


Documentation Impact
====================

The Neutron API reference will need to be updated.


References
==========

.. [1] `ml2_conf.ini`:
       /etc/neutron/plugins/ml2/ml2_conf.ini

.. [2] `OpenStack Networking`:
       https://docs.openstack.org/neutron/latest/admin/intro-os-networking.html

.. [3] `Self-service network`:
       https://docs.openstack.org/newton/install-guide-rdo/launch-instance-networks-selfservice.html

.. [4] `neutron.conf`:
       /etc/neutron/neutron.conf

Related Information
-------------------

Neutron v2 API: https://developer.openstack.org/api-ref/network/v2/
