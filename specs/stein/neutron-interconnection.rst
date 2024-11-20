..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Neutron-Neutron Interconnections
================================

Launchpad RFE: https://bugs.launchpad.net/neutron/+bug/1750368

Problem Description
===================

Today, to realize connectivity between two OpenStack clouds or more (e.g.
between distinct OpenStack deployments, or between OpenStack regions,
for instance) some options are available, such as floating IPs, VPNaaS
(IPSec-based), and BGPVPNs.

However, none of these options are appropriate to address use cases where all
the following properties are desired:

1. interconnection consumable on-demand, without admin intervention [#f1]_

2. have network isolation and allow the use of private IP addressing
   end-to-end [#f2]_

3. avoid the overhead of packet encryption [#f3]_

An additional design requirement to the solution is to avoid introducing a
component that would require having admin rights on all the OpenStack
clouds involved.

The goal of this spec is a solution to provide network connectivity
between two or more OpenStack deployments or regions, respecting these
requirements.

Use cases
---------

Example 1
~~~~~~~~~

User Foo has credentials for OpenStack cloud A and OpenStack cloud B, one
Router X in OpenStack A, one Router Y in OpenStack B, both using a distinct
subnet in the private IPv4 address space.

User Foo would like to consume via an API a service that will result in
establishing IP connectivity between Router A and Router B.

Example 2
~~~~~~~~~

Same as example 1, but this time L2 connectivity is desired between Network X
in OpenStack A, and Network Y in OpenStack B.

Example 3
~~~~~~~~~

Same as example 1 or 2, but with 3 OpenStack deployments involved.

Proposed Change
===============

Overview
--------

The proposition consists in introducing a service plugin and a corresponding
API extension involving a new 'interconnection' resource. The 'interconnection'
resource on a Neutron instance will refer to both a local resource (e.g.
Router A) and a remote resource (OpenStack B:Router B), and will have the
semantic that connectivity is desired between the two.

This resource will be part of two sorts of API calls:

A. calls between the user having the connectivity need and each Neutron
   instance involved; the role of these calls is to let the need for
   connectivity be known by all Neutron instances

B. API calls between a Neutron instance and another Neutron instance; the role
   of these calls is to:

   * let each Neutron instance validate the consistency between the
     "interconnection" resources defined locally and the "interconnection"
     resources made in other Neutron instances

   * after this validation, let two Neutron instances identify the mechanism
     to use and the per-interconnection parameters (depending on the
     mechanism; could be a pair of VLANs on a interconnection box, a BGPVPN
     RT identifiers, VXLAN ids, etc.)

::

                                .-------------.
                                | tenant user |
                                '----+---+----'
                                     |   |
    .--------------------------------'   '--------------------.
    | A1. create "interconnection"                            |
    |     between local net X,                                |
    |     and "Neutron B: net Y"                              |
    |                                          A2. create "interconnection"
    |                                              between local net Y,
    |                                              and "Neutron A: net X"
    |                                                         |
    |                                                         |
    |                                                         |
    |                                                         |
    V                         B1. check symmetric inter.      V
 .-------------------------.                      (fail) .--------------------.
 |                         +---------------------------> |                    |
 |   Neutron A             |                             |   Neutron B        |
 |                         | <---------------------------+                    |
 |                         |  B2. check symmetric inter. |                    |
 |                         |                       (ok!) |                    |
 |                         |                             |                    |
 |                         +---------------------------> |                    |
 |                         | <---------------------------+                    |
 '-------------------------'  B3. exchange info to build '--------------------'
                    net X        interconnection           net Y
                      |                                      |
                      |                                      |
                    --+                                      +--
                      |      C. interconnection is built     |
                      +--    - - - - - - - - - - - - - -   --+
                      |                                      |
                      |                                      |


Note that the order between A1/A2/B1/B2 can vary, but the result is
unchanged: at least one of the two Neutron instances will eventually confirm
that the interconnection has been defined on both side symmetrically, and the
interconnection setup phase will ultimately proceed on both sides (see
:ref:`details`).

When more than two OpenStack deployments, or more than two OpenStack regions,
are involved, these API calls will happen for each pair of region/deployments.

Base assumptions, trust model
-----------------------------

The base assumption underlying the trust model in this proposal is that the end
user requesting connectivity delegates trust to each OpenStack deployment to
provide only the interconnection requested, i.e. to not create connectivity
between resources unless requested.

To respect this contract each OpenStack deployment needs, by definition, a
trust relationship with the other(s) OpenStack deployment(s) involved in these
interconnections; practically speaking it cannot do better, when receiving
a packet from another deployment, identified as intended for own of its local
network A (VLAN, VXLAN ID, MPLS label, etc.) to trust that this identifier was
pushed by the OpenStack deployment by mechanisms ultimately respecting the
contract of these specifications.

Another aspect, obvious but better made explicit, is that the choice and
definition of the network identifiers that will be used for an interconnection
and to keep interconnections isolated from one another, are not controlled
by consumers of this API. In this proposal these consumers do not and cannot
write, or even read, these identifiers.

Note that only the API calls to the 'interconnection' resources at steps A1/A2
require write access to the "interconnection" resources by tenant users (but
not to the attributes related to the network mechanism to use).

The calls at steps B1/B2/B3, only require read-only access to these resources;
this can be achieved by introducing an "interconnection" role with read-only
access to all "interconnection" resources, and having each OpenStack deployment
having credentials for a user with this role in other OpenStack deployments.

With the above in mind, Keystone federation is not required for the calls at
steps A1/A2, nor for the calls at step B1/B2. However, using Keystone
Federation for the user(s) used at step B1/B2 will certainly be useful and
will avoid requiring the management in each Neutron instance of the
credentials to use to each other OpenStack.

Interconnection mechanisms
--------------------------

Although these specifications try to be agnostic to the network technique
ultimately used to realize an interconnection, the assumption is made that
for each 'interconnection', there is a technique common to the two OpenStack
deployments involved.

The approach proposed is a simple approach where each OpenStack deployment
determines based on a configuration file, which technique to use when
establishing an interconnection with a given OpenStack deployment.

Note that only the parameters that do not differ between two interconnection
would sit in a configuration file. The API exchange between two Neutron
instances is used to exchange parameters that are specific to each
interconnection.

Example interconnection techniques
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following techniques can be considered for realizing interconnections:

* BGP-based VPNs: already supported via the Neutron BGPVPN Interconnection
  service (see networking-bgpvpn_), it allows to create L2 (with EVPN) or L3
  connectivity (with BGP/MPLS IP VPNs)

* VXLAN stitching with networking-l2gw_ (details remain to be investigated)

* VLAN stitching (details remain to be investigated)

It is expected that the first implementation will provide at least support
for the BGPVPN interconnection technique, which is already supported across
an interesting range of Neutron backends (Neutron reference drivers,
OpenDaylight, OpenContrail, Nuage Networks), and hence would allow this API
extension to be implemented on day one with a support for all these controllers
without any further per-controller driver development.

.. _details:

Details on operations
---------------------

When an "interconnection" resource is created, the Neutron instance will check
that the symmetric interconnection exists on the remote Neutron instance
designated in the interconnection, and will not proceed further until this
becomes true.

This check is what establishes the end-to-end trust, that on both sides the
connectivity has been requested.

Once a Neutron instance determines that an interconnection is symmetrically
defined, further exchanges happen to determine the network parameters to use
to realize the interconnection:

* the Neutron (e.g. Neutron B) that just confirmed the symmetricity allocates
  the required network identifiers, and asks the remote Neutron instance (A)
  to refresh its state

* Neutron A refreshes its state: checks symmetricity again (which now succeeds)
  retrieves at the same time the network identifiers allocated by B,
  and asks Neutron B to refresh

* Neutron B refreshes again, this time retrieves at the same time the
  network identifiers allocated by A

In the above, "ask the remote Neutron instance to refresh its state" is
done with a PUT on a specific ``refresh`` action on the ``interconnection``.

.. _lifecycle:

Interconnection lifecycle
-------------------------

In the previous section, it is implicit that an ``interconnection`` is along
its life in different states before it is ultimately realized.  When an
``interconnection`` resource is deleted on one side, the other side need also
to ultimately be able to update its own state (if only for cleanup purpose
or giving proper feedback to end users).

Additionally, the interaction between a Neutron instance with another Neutron
instance needs to happen out of the API call processing path, because it is not
desirable that the success of a local API call would depend on the success of
an operation with an external component which possibly would not be available
at the moment.

For all these reasons, a state machine will be introduced to handle the
lifecycle of an interconnection resource, with triggered and periodic operation
being done out-of-band of the API calls, to handle the operations for each
state.

Exposing this state in the API will allow:

* end users to have feedback on how close they are to having something working

* each Neutron instance to possibly identify that the remote state
  is inconsistent with the local state

State machine summary:

TO_VALIDATE
  interconnection resource has been created, but the existence of the
  symmetric interconnection hasn't been validated yet

VALIDATED
  the existence of the symmetric interconnection has been validated and local
  interconnection parameters have been allocated (remote parameters are
  still unknown)

ACTIVE
  both local parameters and remote parameters are known, interconnection has
  been setup, it should work

TEARDOWN
  local action taken to delete this interconnection, action
  is being taken to have the remote state get in sync

(DELETED)
  implicit state corresponding to the resource not existing anymore

.. image:: /images/stein/state-machine-summary.png

REST API Impact
---------------

The proposal is to introduce an API extension ``interconnection``, exposing a
new ``interconnection`` resource.

Interconnection resource
~~~~~~~~~~~~~~~~~~~~~~~~

The new ``interconnection`` API resource will be introduced under the
``interconnection`` API prefix, and having the following attributes:

+-------------------------+--------+----------------+--------------------------------+
|Attribute Name           |Type    |Access          | Comment                        |
+=========================+========+================+================================+
|id                       | uuid   | RO             |                                |
+-------------------------+--------+----------------+--------------------------------+
|project_id               | uuid   | RO             |                                |
+-------------------------+--------+----------------+--------------------------------+
|type                     | enum   | RO             | ``router``, ``network_l2``,    |
|                         |        |                | ``network_l3``                 |
+-------------------------+--------+----------------+--------------------------------+
|state                    | enum   | RO             | see states in :ref:`lifecycle` |
|                         |        | will be updated|                                |
|                         |        | by Neutron     |                                |
|                         |        | along the life |                                |
|                         |        | of the resource|                                |
+-------------------------+--------+----------------+--------------------------------+
|name                     | string | RW             |                                |
+-------------------------+--------+----------------+--------------------------------+
|local_resource_id        | uuid   | RO             | router or network UUID         |
+-------------------------+--------+----------------+--------------------------------+
|remote_resource_id       | uuid   | RO             | router or network UUID         |
+-------------------------+--------+----------------+--------------------------------+
|remote_keystone          | string | RO             | AUTH_URL of remote             |
|                         |        |                | keystone                       |
+-------------------------+--------+----------------+--------------------------------+
|remote_region            | string | RO             | region in remote keystone      |
+-------------------------+--------+----------------+--------------------------------+
|remote_interconnection_id| uuid   | RO             | uuid of remote interconnection |
+-------------------------+--------+                +--------------------------------+
|local_parameters         | dict   | will be updated|                                |
+-------------------------+--------+ by Neutron     |                                |
|remote_parameters        | dict   | along the life |                                |
|                         |        | of the resource|                                |
+-------------------------+--------+----------------+--------------------------------+

This resource will be used with typical CRUD operations:

* ``POST /v2.0/interconnection/interconnections``

* ``GET /v2.0/interconnection/interconnections``

* ``GET /v2.0/interconnection/interconnections/<uuid>``

* ``PUT /v2.0/interconnection/interconnections/<uuid>``

* ``DELETE /v2.0/interconnection/interconnections/<uuid>``

Additionally, an additional REST operation is introduced to trigger a
``refresh`` action on an ``interconnection`` resource:

* ``PUT /v2.0/interconnection/interconnections/<uuid>/refresh``

When this action is triggered the neutron instance on which the call is made
will try to retrieve (``GET``) an ``interconnection`` resource on the remote
neutron instance having same ``type`` as the local resource,
as ``local_resource_id`` the local resource ``remote_resource_id``, and
as ``remote_resource_id`` the local resource ``local_resource_id``.  Depending
on the current local state, and depending on success or failure to find such
a resource, the local state machine will transition.

Example
~~~~~~~

This shows an example of API exchanges for a Neutron-Neutron interconnection
between two Networks.

API Call A1, from tenant user to Neutron A::

    POST /v2.0/interconnection/interconnections
         {'interconnection': {
             'type': 'network_l3',
             'local_resource_id': <uuid of network X>,
             'remote_keystone': 'http//<keystone-B>/identity',
             'remote_region': 'RegionOne',
             'remote_resource_id': <uuid of network Y>
          }}

    Response: 200 OK

    {'interconnection': {
         'id': <uuid 1>,
         ...
     }}

API Call B1, from Neutron A to Neutron B::

    GET /v2.0/interconnection/interconnections?local_resource_id=<uuid of network Y>&remote_resource_id=<uuid of network X>

    Response: 404 Not Found

API Call A2, from tenant user to Neutron B::

    POST /v2.0/interconnection/interconnections
         {'interconnection': {
             'type': 'network_l3',
             'local_resource_id': <uuid of network Y>,
             'remote_keystone': 'http//<keystone-A>/identity',
             'remote_region': 'RegionOne',
             'remote_resource_id': <uuid of network X>
          }}

    Response: 200 OK

    {'interconnection': {
         'id': <uuid 2>,
         ...
     }}

API Call B2, from Neutron B to Neutron A::

    GET /v2.0/interconnection/interconnections/local_resource_id=<uuid of network X>&remote_resource_id=<uuid of network Y>

    Response: 200 OK

    {'interconnection': {
         'id': <uuid 1>,
         ...,
         'local_parameters': {}
     }}

API Call B3' from Neutron B to Neutron A::

    PUT /v2.0/interconnection/interconnections/<uuid 1>/refresh

    Response: 200 OK

API Call B3'', from Neutron A to Neutron B ::

    GET /v2.0/interconnection/interconnections/local_resource_id=<uuid of network Y>&remote_resource_id=<uuid of network X>

    Response: 200 OK

    {'interconnection': {
         'id': <uuid 2>,
         ...,
         'local_parameters': {
             'foo': '42'
         }
     }}

API Call B3''', from Neutron A to Neutron B ::

    PUT /v2.0/interconnection/interconnections/<uuid 2>/refresh

    Response: 200 OK

API Call B3'''', from Neutron B to Neutron A ::

    GET /v2.0/interconnection/interconnections/<uuid 1>

    Response: 200 OK

    {'interconnection': {
         'id': <uuid 1>,
         ...,
         'remote_interconnection_id': <uuid 2>,
         'remote_parameters': {
             'foo': '42'
         },
         'local_parameters': {
             'bar': '43'
         }
     }}


Command Line Client Impact
--------------------------

python-neutronclient will be updated to introduce an OSC extension to
create/remove/update/delete ``interconnection`` resources.

Client Libraries Impact
--------------------------

python-neutronclient and openstacksdk will need to be updated to support
create/remove/update/delete operations on ``interconnection`` resources.

Credits
-------

Przemyslaw Jasek contributed to exploring ideas that lead to this proposal
during a six-months internship at Orange.


References
==========

-  OpenStack Summit Sydney, lightning talk
   https://www.openstack.org/videos/sydney-2017/neutron-neutron-interconnections

.. _networking-bgpvpn: https://docs.openstack.org/networking-bgpvpn
.. _networking-l2gw: https://docs.openstack.org/networking-l2gw
.. [#f1] possible with floating IPs, VPNaaS, but not with the BGP VPN
   interconnections API extension (using a BGPVPN does not require admin
   right, but creating a new BGPVPN does require admin rights)
.. [#f2] possible with VPNaaS, and BGP VPN interconnections, but not with
   floating IPs
.. [#f3] possible with floating IPs and BGP VPN interconnections, but by
   definition not with VPNaaS
