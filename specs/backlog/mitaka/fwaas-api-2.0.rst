..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Firewall as a Service API 2.0
==========================================

**Launchpad blueprint:**

| https://blueprints.launchpad.net/neutron/+spec/fwaas-api-2.0

This spec introduces a few enhancements to Firewall as a Service (FWaaS)
API including making it more granular by giving the users the ability to
apply the firewall rules at the port level rather than at the router
level. Support is extended to various types of Neutron ports, including
VM ports and SFC ports as well as router ports. It also aims to provide
better grouping mechanisms (firewall groups, address groups and service
groups) and discuss the use of a common classifier in achieving it. While
this spec is proposing the API, the implementation will be carried out in
phases subject to availability of people to do the work. And at the end of
Mitaka - we will have a working reference implementation but, that may
not embody all the enhancements in this spec. The implementation phases
will be tracked by an appropriate RFE referencing this as the parent spec.

Problem Description
===================

The Security Group API extension is a well known API for handling network
traffic. For basic traffic filtering, the Security Group API is well
understood by users, but as cloud infrastructure gains wider acceptance
into the enterprise, the security group API, which was built for public
cloud infrastructure becomes insufficient for the security and network
environments inside the enterprise.

As such, one-off changes to the Security Group API end up occurring in
each deployment of OpenStack. One solution is to allow deployers of
OpenStack to "extend" an API extension to make it fit in their security
and network environment. Ideally, the Firewall as a Service API is the
API where usecases more advanced than the basic "let any traffic from X
IP into Y port into my group of VMs" should be supported.

But the existing Firewall as a Service API has been deprecated in Liberty
because it was fairly limited, requiring substantial enhancements to make
it cover all the use cases raised by the community. There are also some
overlapping functionalities between FWaaS API and the SG API which need
to be rationalized.

| Use Cases:

https://trello.com/b/TIWf4dBJ/fwaas-usecase-categorization


Proposed Change
===============

It is proposed to harmonize the FWaaS and Security Group models by
converging the implementation of FWaaS and Security Groups but keeping
a separate API for each of them while relying on a common backend. This
spec proposes an enhanced FWaaS API that incorporates Security Groups
functionality such that the FWaaS API becomes a superset of what is
exposed by the Security Group API.

Relative to the FWaaS 1.0 API, the FWaaS 2.0 API provides the following
enhancements:

    * Applies at the granularity of Neutron ports rather than tenant
      wide or a set of routers in a tenant.

    * Applies to various types of Neutron ports, including VM ports and
      SFC ports as well as router ports. By applying FWaaS at VM ports,
      it will be possible to filter east/west intra-subnet traffic as
      well as east/west inter-subnet traffic and north/south traffic.
      This will also allow FWaaS to filter east/west traffic when DVR is
      used, as opposed to FWaaS 1.0 that only filters north/south
      traffic in case of DVR [4], [5].

    * Allows for different firewall policies with different firewall
      rules to be applied to different directions (ingress vs egress).

    * Introduces the Firewall Group construct for Neutron ports. This
      becomes the association point for binding firewall policies and
      Neutron ports, as well as a way to specify allowed sources and
      destinations in firewall rules. This reduces or eliminates the
      need to specify IP addresses for east/west traffic flows in
      firewall rules. For example, when Firewall Group A is used as the
      destination firewall group id in a rule within a policy in Firewall
      Group B, we care only about the ports associated with Firewall
      Group A. When a packet arrives, it is matched if its destination
      ip address matches an IP address of one the ports in Firewall
      Group A.

    * Adds indirections through address group and service group that
      allow groups of addresses and groups of L4 ports to be specified
      once and reused in multiple rules and multiple firewall policies.

    * Allows multiple firewall group associations for the same Neutron
      port. For example, one firewall group applied to a tenant's web
      servers may specify a firewall policy that restricts traffic to
      HTTP only (intended for north/south traffic), while another
      firewall group applied to all the tenant's instances may specify a
      firewall policy that allows all traffic types between sources and
      destinations in that firewall group (east/west traffic).


Relative to the Security Groups API [7], the FWaaS 2.0 API provides the
following enhancements:

    * Adds an explicit action attribute to rules so that "deny" and
      "reject" actions can be specified in addition to the existing
      "allow" action. This is particularly important for tenant or
      service provider network admins that specify firewall policies
      meant to apply to all of a tenant's or service provider's
      instances, regardless of application.

    * Allows filtering based on both source and destination address
      prefixes, rather than just the remote (source for ingress traffic,
      destination for egress traffic) address prefixes.

    * Allows filtering based on both source and destination L4 port
      ranges, rather than just destination L4 port range.

    * By adding indirections through firewall group, address group and
      service group, allows for operational separation of responsibilities
      between users and experts such as network admins. Experts can define
      groups of IP address prefixes in address groups, and define service
      groups using protocol and source/destination ports. Users can then
      easily define firewall rules that refer to firewall group and
      service group, without having to know about IP addresses and L4
      ports. For example, a service group named *All Web Server Ports*
      could be defined with protocol TCP and destination L4 ports 80,
      8080, and 443.

    * Adds a "description" attribute to firewall rules.

    * Adds an "admin status" attribute to firewall rules.

    * Adds a "public" attribute allowing sharing of firewall rules
      between different projects.

    * Firewall groups reference firewall rules through a firewall
      policy. In particular, this allows reuse of sharable firewall
      policies that are referenced by multiple firewall groups.

    * In the future, it is expected that service groups will support
      deep packet inspection, so that traffic can be matched based on an
      application ID and L7 fields such as URL strings, instead of or in
      addition to L4 port ranges.


Relationship Between FWaaS and Security Groups
----------------------------------------------

FWaaS and Security Groups remain as separate features. The Security Group
API will be retained in addition to the FWaaS 2.0 API for two reasons.

The first reason to retain the existing Security Group API is for those
users who want to leverage existing functionality in a way that aligns
as closely as possible with non-OpenStack security group definitions.

The second reason has to do with "defense in depth", where multiple
layers of data plane access control are defined and applied. With
existing FWaaS 1.0 and Security Groups, the perimeter firewall
functionality is defined in FWaaS and enforced at OpenStack routers,
while application firewall functionality is defined through Security
Groups and enforced at VM ports. Both north/south and inter-subnet
traffic must be allowed by both FWaaS and Security Groups in order to
pass end-to-end from the source to the destination.

With FWaaS 2.0, it is important to retain "defense in depth" even when
FWaaS is enforced at VM ports. When both FWaaS and Security Groups are
associated with the same Neutron port, a packet must be allowed by both
features, i.e. "deny" wins between FWaaS and Security Groups. This
behavior is adopted to address typical use cases where a tenant network
admin uses FWaaS to specify tenant wide rules that are to be applied
regardless of the application, while an application deployer uses
Security Groups to narrow down allowed traffic to only what is needed
for a specific application.

For example, a network admin creates a firewall rule that denies port
25 traffic. Even if the application deployer creates a security group
rule that allows port 25 traffic, the port 25 traffic will be denied.

Note that as with the existing FWaaS 1.0 API and Security Groups, by
default OpenStack policy does not distinguish between different roles
within a project. Default OpenStack policy will not prevent different
users from the same project (e.g. application deployers vs tenant
network admins) from accessing the FWaaS API. This may be investigated
in future phases.

In future phases, the FWaaS 2.0 API will be enhanced so that multiple
layers of "defense in depth" can be defined using only the FWaaS 2.0
API. This will allow application deployers to take advantage of the
enhancements of FWaaS 2.0 relative to Security Groups, while retaining
"defense in depth". This will also allow for more than 2 layers of
"defense in depth", for example tenant application deployers, tenant
network admins, and service provider network admins.


REST API Impact
---------------

Firewall Address Groups
~~~~~~~~~~~~~~~~~~~~~~~~~

+-------------------+---------+-------+------+---------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                           |
+===================+=========+=======+======+=======================================+
| id                | uuid-str| N/A   | R    | Unique identifier for the             |
|                   |         |       |      | address_group object.                 |
+-------------------+---------+-------+------+---------------------------------------+
| name              | String  | No    | CRU  | Human readable name for the address   |
|                   |         |       |      | group (255 characters limit). Does not|
|                   |         |       |      | have to be unique.                    |
+-------------------+---------+-------+------+---------------------------------------+
| description       | String  | No    | CRU  | Human readable description for the    |
|                   |         |       |      | address group (255 characters limit). |
+-------------------+---------+-------+------+---------------------------------------+
| project_id        | uuid-str| Yes   | CR   | Owner of the address group. Only      |
|                   |         |       |      | admin users can specify a project     |
|                   |         |       |      | identifier other than their own.      |
+-------------------+---------+-------+------+---------------------------------------+
| cidrs             | List    | Yes   | CRU  | Array of key-value pairs of cidr and  |
|                   |         |       |      | ip version.                           |
+-------------------+---------+-------+------+---------------------------------------+

|
|

Firewall Rules
~~~~~~~~~~~~~~~

Note that as with FWaaS 1.0, in FWaaS 2.0 firewall rules always use stateful connection
tracking.

+------------------------+------------+-----+------+---------------------------------------+
| Attribute              | Type       | Req | CRUD |  Description                          |
+========================+============+=====+======+=======================================+
| id                     | uuid-str   | N/A | R    | Unique identifier for the firewall    |
|                        |            |     |      | rule object.                          |
+------------------------+------------+-----+------+---------------------------------------+
| project_id             | uuid-str   | Yes | CRU  | Owner of the firewall rule. Only      |
|                        |            |     |      | admin users can specify a project     |
|                        |            |     |      | identifier other than their own.      |
+------------------------+------------+-----+------+---------------------------------------+
| name                   | String     | No  | CRU  | Human readable name for the firewall  |
|                        |            |     |      | rule (255 characters limit). Does     |
|                        |            |     |      | not have to be unique.                |
+------------------------+------------+-----+------+---------------------------------------+
| description            | String     | No  | CRU  | Human readable description for the    |
|                        |            |     |      | firewall Rule (255 characters limit). |
+------------------------+------------+-----+------+---------------------------------------+
| public                 | Bool       | No  | CRU  | When set to True makes this firewall  |
|                        |            |     |      | rule visible to projects other than   |
|                        |            |     |      | its owner, and can be used in         |
|                        |            |     |      | firewall policies not owned by its    |
|                        |            |     |      | project.                              |
+------------------------+------------+-----+------+---------------------------------------+
| protocol               | String     | No  | CRU  | IP Protocol.                          |
+------------------------+------------+-----+------+---------------------------------------+
| source_port            | port-range | No  | CRU  | Source port number or a range (an     |
|                        |            |     |      | int in [1, 65535] or range in a:b).   |
+------------------------+------------+-----+------+---------------------------------------+
| destination_port       | port-range | No  | CRU  | Destination port number or a range (  |
|                        |            |     |      | an int in [1, 65535] or range in a:b).|
+------------------------+------------+-----+------+---------------------------------------+
| service_group_id       | uuid-str   | No  | CRU  | UUID of the service group [6].        |
+------------------------+------------+-----+------+---------------------------------------+
| ip_version             | Integer    | No  | CRU  | IP Protocol Version.                  |
+------------------------+------------+-----+------+---------------------------------------+
| source_ip_address      | String     | No  | CRU  | Source IP address or CIDR.            |
+------------------------+------------+-----+------+---------------------------------------+
| destination_ip_address | String     | No  | CRU  | Destination IP address or CIDR.       |
+------------------------+------------+-----+------+---------------------------------------+
| source_address         | uuid-str   | No  | CRU  | When a source_address_group is        |
| _group_id              |            |     |      | specified, it is matched when the     |
|                        |            |     |      | source IP address in the packet       |
|                        |            |     |      | matches one of the IP addresses in    |
|                        |            |     |      | the address group.                    |
+------------------------+------------+-----+------+---------------------------------------+
| destination_address    | uuid-str   | No  | CRU  | When a destination_address_group is   |
| _group_id              |            |     |      | specified, it is matched when the     |
|                        |            |     |      | destination IP address in the packet  |
|                        |            |     |      | matches one of the IP addresses in the|
|                        |            |     |      | address group.                        |
+------------------------+------------+-----+------+---------------------------------------+
| source_firewall_group  | uuid-str   | No  | CRU  | When a source_firewall_group is       |
| _id                    |            |     |      | specified, it is matched when the     |
|                        |            |     |      | source IP address in the packet       |
|                        |            |     |      | matches an IP address of one of the   |
|                        |            |     |      | ports in the firewall group.          |
|                        |            |     |      | Note:  This holds true when firewall  |
|                        |            |     |      | group contains a list of vm ports.    |
+------------------------+------------+-----+------+---------------------------------------+
| destination_firewall   | uuid-str   | No  | CRU  | When a destination_firewall_group is  |
| _group_id              |            |     |      | specified, it is matched when the     |
|                        |            |     |      | destination IP address in the packet  |
|                        |            |     |      | matches an IP address of one of the   |
|                        |            |     |      | ports in the firewall group.          |
|                        |            |     |      | Note:  This holds true when firewall  |
|                        |            |     |      | group contains a list of vm ports.    |
+------------------------+------------+-----+------+---------------------------------------+
| action                 | String     | No  | CRU  | Action to be performed on the         |
|                        |            |     |      | traffic matching the rule (ALLOW,     |
|                        |            |     |      | DENY, REJECT). Default: DENY.         |
+------------------------+------------+-----+------+---------------------------------------+
| enabled                | Bool       | No  | CRU  | When set to False will disable this   |
|                        |            |     |      | rule in the firewall policy.          |
|                        |            |     |      | Facilitates selectively turning off   |
|                        |            |     |      | rules without having to disassociate  |
|                        |            |     |      | the rule from the firewall policy.    |
|                        |            |     |      | Default: True.                        |
+------------------------+------------+-----+------+---------------------------------------+

|

Note: At most one of source_ip_address, source_address_group_id and
source_firewall_group_id can be specified.  The rule is matched when the
source IP address in the packet matches any one of: source_ip_address,
one of the IP addresses in the address group, or an IP address of one
of the ports in the firewall group. If you want it to match any packet,
set the source or destination to 0.0.0.0/0 or ::/0. The same applies to
destination_ip_address, destination_address_group_id, and destination
_firewall_group_id, with respect to the destination IP address in the
packet.

|

Firewall policies
~~~~~~~~~~~~~~~~~~

+----------------+------------+-----+------+-----------------------------------------+
| Attribute      | Type       | Req | CRUD | Description                             |
+================+============+=====+======+=========================================+
| id             | uuid-str   | N/A | R    | Unique identifier for the firewall      |
|                |            |     |      | policy object.                          |
+----------------+------------+-----+------+-----------------------------------------+
| project_id     | uuid-str   | Yes | CR   | Owner of the firewall policy. Only      |
|                |            |     |      | admin users can specify a project       |
|                |            |     |      | identifier other than their own.        |
+----------------+------------+-----+------+-----------------------------------------+
| name           | String     | No  | CRU  | Human readable name for the firewall    |
|                |            |     |      | policy (255 characters limit). Does     |
|                |            |     |      | not have to be unique.                  |
+----------------+------------+-----+------+-----------------------------------------+
| description    | String     | No  | CRU  | Human readable description for the      |
|                |            |     |      | firewall Policy (255 characters limit). |
+----------------+------------+-----+------+-----------------------------------------+
| firewall_rules | List       | No  | CRU  | This is an ordered list of firewall     |
|                |            |     |      | rule uuids. The firewall applies the    |
|                |            |     |      | rules in the order in which they appear.|
+----------------+------------+-----+------+-----------------------------------------+
| audited        | Bool       | No  | CRU  | When set to True by the policy owner    |
|                |            |     |      | indicates that the firewall policy has  |
|                |            |     |      | been audited. Each time the firewall    |
|                |            |     |      | policy or the associated firewall       |
|                |            |     |      | rules are changed, this attribute will  |
|                |            |     |      | be set to False and will have to be     |
|                |            |     |      | explicitly set to True through an       |
|                |            |     |      | update operation.                       |
+----------------+------------+-----+------+-----------------------------------------+
| public         | Bool       | No  | CRU  | When set to True makes this firewall    |
|                |            |     |      | policy visible to projects other than   |
|                |            |     |      | its owner.                              |
+----------------+------------+-----+------+-----------------------------------------+

|

Firewall groups
~~~~~~~~~~~~~~~~

Firewall Groups (similar to Security Groups) are the central construct
of the FWaaS 2.0 API. They serve two purposes:

    1. Through firewall group / port associations, they specify the
       the Neutron ports that are the points of enforcement of firewall
       policies.

    2. Through the source_firewall_group_id and destination_firewall
       _group_id in firewall rules, they allow for filtering based on
       source and destination identities, while minimizing the need to
       specify long lists of IP addresses.

       For each source_firewall_group and destination_firewall_group,
       the OpenStack controller will tell OpenStack FWaaS agents the set
       of IP addresses for all VM ports associated with firewall group.
       The list of router ports associated with the firewall group will
       be passed as is.

Similar to Security Groups, for each project, one Firewall Group named
"default" will be created automatically. This default Firewall Group
will be applied to all new VM ports within that project, unless it is
explicitly disassociated from the new VM port. This provides a way for a
tenant network admin to define a tenant wide firewall policy that
applies to all VM ports, except when explicitly provisioned otherwise.
The default firewall rules for the default Firewall Group are allow all,
i.e. the tenant network admin will have to explicitly define firewall
policies and rules in order for the default Firewall Group to take
effect. For example, the tenant network admin may want to deny
connectivity to certain IP addresses known to be harmful, or deny use of
particular L4 ports. This behavior is chosen assuming that typical
deployments will use "defense in depth", with application deployers
specifying default Security Groups, while tenant network admins specify
default Firewall Groups.

|

+-------------------+---------+-------+------+---------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                           |
+===================+=========+=======+======+=======================================+
| id                | uuid-str| N/A   | R    | Unique identifier for the firewall    |
|                   |         |       |      | group object.                         |
+-------------------+---------+-------+------+---------------------------------------+
| name              | string  | No    | CRU  | Human readable name for the firewall  |
|                   |         |       |      | group (255 characters limit). Does    |
|                   |         |       |      | not have to be unique.                |
+-------------------+---------+-------+------+---------------------------------------+
| description       | string  | No    | CRU  | Human readable description for the    |
|                   |         |       |      | firewall group (255 characters limit).|
+-------------------+---------+-------+------+---------------------------------------+
| project_id        | uuid-str| Yes   | CR   | Owner of the firewall group. Only     |
|                   |         |       |      | admin users can specify a project     |
|                   |         |       |      | identifier other than their own.      |
|                   |         |       |      | Default: derived from authentication  |
|                   |         |       |      | token.                                |
+-------------------+---------+-------+------+---------------------------------------+
| ingress_firewall  | uuid-str| No    | CRU  | 'null' if not associated with any     |
| _policy_id        |         |       |      | firewall policy.                      |
+-------------------+---------+-------+------+---------------------------------------+
| egress_firewall   | uuid-str| No    | CRU  | 'null' if not associated with any     |
| _policy_id        |         |       |      | firewall policy.                      |
+-------------------+---------+-------+------+---------------------------------------+
| ports             | List    | No    | CRU  | List of port_ids that will be         |
|                   |         |       |      | associated to this firewall_group.    |
+-------------------+---------+-------+------+---------------------------------------+


List address groups
^^^^^^^^^^^^^^^^^^^^^

Lists address groups.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/address_groups``                         |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401)                    |
    +----------------+---------+--------------------------------------+

|

**Example List address groups: JSON request**

.. code::

    GET /v2.0/fw/address_groups.json
    User-Agent: python-neutronclient
    Accept: application/json

**Example List address groups: JSON response**


.. code::

    {
        "address_groups": [
            {
                "description": "",
                "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
                "name": "ADDR_GP_1",
                "project_id": "45977fa2dbd7482098dd68d0d8970117",
                "cidrs": [
                   {"cidr": "132.168.4.12/24", "ip_version": 4},
                   {"cidr": "2001::db8::f00/64", "ip_version": 6}
                ]
            }
        ]
    }

Show address group details
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Shows address group details.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/address_groups/<address_group_id>``      |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Forbidden(403), \ |
    |                |         | Not Found (404)                      |
    +----------------+---------+--------------------------------------+

|

**Example Show address group: JSON request**

.. code::

    GET /v2.0/fw/address_groups/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json


**Example Show address group: JSON response**

.. code::

    {
       "address_group": {
            "description": "",
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "ADDR_GP_1",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "cidrs": [
               {"cidr": "132.168.4.12/24", "ip_version": 4},
               {"cidr": "2001::db8::f00/64", "ip_version": 6}
            ]
        }
    }



Create address group
^^^^^^^^^^^^^^^^^^^^^

Creates an address group.

    +----------------+------------------------------------------------+
    | Request Type   | ``POST``                                       |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/address_groups/``                        |
    +----------------+---------+--------------------------------------+
    |                | Success | 201                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400)  |
    +----------------+---------+--------------------------------------+

|

**Example Create address group: JSON request**

.. code::

    POST /v2.0/fw/address_groups.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "address_group": {
            "name": "ADDR_GP_1",
            "cidrs": [
               {"cidr": "132.168.4.12/24", "ip_version": 4},
               {"cidr": "2001::db8::f00/64", "ip_version": 6}
            ]
        }
    }

**Example Create address group: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
       "address_group": {
            "description": "",
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "ADDR_GP_1",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "cidrs": [
               {"cidr": "132.168.4.12/24", "ip_version": 4},
               {"cidr": "2001::db8::f00/64", "ip_version": 6}
            ]
        }
    }


Update address group
^^^^^^^^^^^^^^^^^^^^^

Updates an address group.

    +----------------+------------------------------------------------+
    | Request Type   | ``PUT``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/address_groups/<address_group_id>``      |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400) \|
    |                |         | Not Found(404)                       |
    +----------------+---------+--------------------------------------+

|

**Example Update address group: JSON request**

.. code::

    PUT /v2.0/fw/address_groups/41bfef97-af4e-4f6b-a5d3-4678859d2485.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "address_group": {
            "cidrs": [
               {"cidr": "132.168.4.12/24", "ip_version": 4},
               {"cidr": "2001::db8::f00/64", "ip_version": 6}
            ]
        }
    }


**Example Update address group: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
       "address_group": {
            "description": "",
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "ADDR_GP_1",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "cidrs": [
               {"cidr": "132.168.4.12/24", "ip_version": 4},
               {"cidr": "2001::db8::f00/64", "ip_version": 6}
            ]

        }
    }


Delete address group
^^^^^^^^^^^^^^^^^^^^^

Deletes an address group.

This operation does not return a response body.

    +----------------+------------------------------------------------+
    | Request Type   | ``DELETE``                                     |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/address_groups/<address_group_id>``      |
    +----------------+---------+--------------------------------------+
    |                | Success | 204                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Not Found(404)    |
    |                |         | Conflict(409) The Conflict error     |
    |                |         | response is returned when an         |
    |                |         | operation is performed while         |
    |                |         | address group is in use.             |
    +----------------+---------+--------------------------------------+

|

**Example Delete address group: JSON request**

.. code::

    DELETE /v2.0/fw/address_groups/1be5e5f7-c45e-49ba-85da-156575b60d50.json
    User-Agent: python-neutronclient
    Accept: application/json

**Example Delete address group: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0


List firewall rules
^^^^^^^^^^^^^^^^^^^^

Lists firewall rules.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_rules``                         |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401)                    |
    +----------------+---------+--------------------------------------+

|

**Example List firewall rules: JSON request**

.. code::

    GET /v2.0/fw/firewall_rules.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example List firewall rules: JSON response**


.. code::

    {
        "firewall_rules": [
            {
                "action": "ALLOW",
                "description": "",
                "service_group_id":"fe99d33c1-b472-44f9-8226-30dc4ffd45332",
                "enabled": true,
                "firewall_policy_id": "asd435dg3-b472-44f9-8226-30dc4ffd45332",
                "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
                "name": "ALLOW_HTTP",
                "position": 1,
                "public": false,
                "protocol": "tcp",
                "source_port": null,
                "destination_port": null,
                "ip_version": 4,
                "source_ip_address": null,
                "destination_ip_address": null
                "source_address_group_id": null,
                "destination_address_group_id": null,
                "source_firewall_group_id": "ds876h5t1-b472-44f9-8226-3087j9u953gh2",
                "destination_firewall_group_id": "f98o6h5t1-b472-44f9-8226-3087j9u953gh2",
                "project_id": "45977fa2dbd7482098dd68d0d8970117"
            }
        ]
    }

Show firewall rule details
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Shows firewall rule details.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_rules/<firewall_rule_id>``      |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Forbidden(403), \ |
    |                |         | Not Found (404)                      |
    +----------------+---------+--------------------------------------+

|

**Example Show firewall rule: JSON request**

.. code::

    GET /v2.0/fw/firewall_rules/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json


**Example Show firewall rule: JSON response**

.. code::

    {
        "firewall_rule": {
            "action": "ALLOW",
            "description": "",
            "service_group_id":"fe99d33c1-b472-44f9-8226-30dc4ffd45332",
            "enabled": true,
            "firewall_policy_id": "asd435dg3-b472-44f9-8226-30dc4ffd45332",
            "id": "9faaf49f-dd89-4e39-a8c6-101839aa49bc",
            "name": "ALLOW_HTTP",
            "position": 1,
            "public": false,
            "protocol": "tcp",
            "source_port": null,
            "destination_port": null,
            "ip_version": 4,
            "source_ip_address": null,
            "destination_ip_address": null,
            "source_address_group_id": null,
            "destination_address_group_id": "f9876h5t1-b472-44f9-8226-3087j9u953gh2",
            "source_firewall_group_id": "ds876h5t1-b472-44f9-8226-3087j9u953gh2",
            "destination_firewall_group_id": null,
            "project_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Create firewall rule
^^^^^^^^^^^^^^^^^^^^^

Creates a firewall rule.

    +----------------+------------------------------------------------+
    | Request Type   | ``POST``                                       |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_rules/``                        |
    +----------------+---------+--------------------------------------+
    |                | Success | 201                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400)  |
    +----------------+---------+--------------------------------------+

|

**Example Create firewall rule: JSON request**

.. code::

    POST /v2.0/fw/firewall_rules.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule": {
            "action": "ALLOW",
            "destination_address_group_id": "f9876h5t1-b472-44f9-8226-3087j9u953gh2"
            "service_group_id": "d2876h5t1-b472-44f9-8245-308dr4u953gh2"
            "enabled": true,
            "name": "ALLOW_HTTP"
        }
    }

**Example Create firewall rule: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall_rule": {
            "action": "ALLOW",
            "description": "",
            "service_group_id": "d2876h5t1-b472-44f9-8245-308dr4u953gh2"
            "enabled": true,
            "firewall_policy_id": null,
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "ALLOW_HTTP",
            "position": 1,
            "public": false,
            "protocol": "tcp",
            "source_port": null,
            "destination_port": null,
            "ip_version": 4,
            "source_ip_address": null,
            "destination_ip_address": null,
            "source_address_group_id": null,
            "destination_address_group_id": "f9876h5t1-b472-44f9-8226-3087j9u953gh2",
            "source_firewall_group_id": "ds876h5t1-b472-44f9-8226-3087j9u953gh2",
            "destination_firewall_group_id": null,
            "project_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }


Update firewall rule
^^^^^^^^^^^^^^^^^^^^^

Updates a firewall rule.

    +----------------+------------------------------------------------+
    | Request Type   | ``PUT``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_rules/<firewall_rule_id>``      |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400) \|
    |                |         | Not Found(404)                       |
    +----------------+---------+--------------------------------------+

|

**Example Update firewall rule: JSON request**

.. code::

    PUT /v2.0/fw/firewall_rules/41bfef97-af4e-4f6b-a5d3-4678859d2485.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule": {
            "public": "true"
        }
    }

**Example Update firewall rule: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::


    {
        "firewall_rule": {
            "action": "ALLOW",
            "description": "",
            "service_group_id": "d2876h5t1-b472-44f9-8245-308dr4u953gh2"
            "enabled": true,
            "firewall_policy_id": null,
            "id": "41bfef97-af4e-4f6b-a5d3-4678859d2485",
            "name": "ALLOW_HTTP",
            "position": 1,
            "public": true,
            "protocol": "tcp",
            "source_port": null,
            "destination_port": null,
            "ip_version": 4,
            "source_ip_address": null,
            "destination_ip_address": null,
            "source_address_group_id": null,
            "destination_address_group_id": "f9876h5t1-b472-44f9-8226-3087j9u953gh2",
            "source_firewall_group_id": "ds876h5t1-b472-44f9-8226-3087j9u953gh2",
            "destination_firewall_group_id": null,
            "project_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }


|

Delete firewall rule
^^^^^^^^^^^^^^^^^^^^^

Deletes a firewall rule.

This operation does not return a response body.

    +----------------+------------------------------------------------+
    | Request Type   | ``DELETE``                                     |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_rules/<firewall_rule_id>``      |
    +----------------+---------+--------------------------------------+
    |                | Success | 204                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Not Found(404)    |
    |                |         | Conflict(409) The Conflict error     |
    |                |         | response is returned when an         |
    |                |         | operation is performed while         |
    |                |         | firewall rule is in use.             |
    +----------------+---------+--------------------------------------+

|

**Example Delete firewall rule: JSON request**

.. code::

    DELETE /v2.0/fw/firewall_rules/1be5e5f7-c45e-49ba-85da-156575b60d50.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Delete firewall rule: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0


List firewall policies
^^^^^^^^^^^^^^^^^^^^^^^

Lists firewall policies.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_policies``                      |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Forbidden(403)    |
    +----------------+---------+--------------------------------------+

|

**Example List firewall policies: JSON request**

.. code::

    GET /v2.0/fw/firewall_policies.json
    User-Agent: python-neutronclient
    Accept: application/json

**Example List firewall policies: JSON response**

.. code::

    {
        "firewall_policies": [
            {
                "audited": false,
                "description": "",
                "firewall_rules": [
                    "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
                ],
                "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
                "name": "test-policy",
                "public": false,
                "project_id": "45977fa2dbd7482098dd68d0d8970117"
            }
        ]
    }


Show firewall policy details
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Shows firewall policy details.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_policies/<firewall_policy_id>`` |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Not Found(404)    |
    +----------------+---------+--------------------------------------+

|

**Example Show firewall policy: JSON request**

.. code::

    GET /v2.0/fw/firewall_policies/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Show firewall policy: JSON response**

.. code::

    {
        "firewall_policy": {
            "audited": false,
            "description": "",
            "firewall_rules": [
                "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
            ],
            "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
            "name": "test-policy",
            "public": false,
            "project_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }


Create firewall policy
^^^^^^^^^^^^^^^^^^^^^^^

Creates a firewall policy.

    +----------------+------------------------------------------------+
    | Request Type   | ``POST``                                       |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_policies``                      |
    +----------------+---------+--------------------------------------+
    |                | Success | 201                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401)                    |
    +----------------+---------+--------------------------------------+

|

**Example Create firewall policy: JSON request**

.. code::

    POST /v2.0/fw/firewall_policies.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_policy": {
            "firewall_rules": [
                "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
            ],
            "name": "test-policy"
        }
    }

**Example Create firewall policy: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall_policy": {
            "audited": false,
            "description": "",
            "firewall_rules": [
                "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
            ],
            "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
            "name": "test-policy",
            "public": false,
            "project_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Update firewall policy
^^^^^^^^^^^^^^^^^^^^^^^

Updates a firewall policy.

    +----------------+------------------------------------------------+
    | Request Type   | ``PUT``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_policies/<firewall_policy_id>`` |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Not Found (404)   |
    +----------------+---------+--------------------------------------+

|

**Example Update firewall policy: JSON request**

.. code::

    PUT /v2.0/fw/firewall_policies/41bfef97-af4e-4f6b-a5d3-4678859d2485.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_policy": {
            "firewall_rules": [
                "a08ef905-0ff6-4784-8374-175fffe7dade",
                "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
            ]
        }
    }

**Example Update firewall policy: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall_policy": {
            "audited": false,
            "description": "",
            "firewall_rules": [
                "a08ef905-0ff6-4784-8374-175fffe7dade",
                "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
            ],
            "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
            "name": "test-policy",
            "public": false,
            "project_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Delete firewall policy
^^^^^^^^^^^^^^^^^^^^^^^

Deletes a firewall policy.

    +----------------+------------------------------------------------+
    | Request Type   | ``DELETE``                                     |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_policies/<firewall_policy_id>`` |
    +----------------+---------+--------------------------------------+
    |                | Success | 204                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Not Found(404)    |
    |                |         | Conflict(409) The Conflict error     |
    |                |         | response is returned when an         |
    |                |         | operation is performed while the     |
    |                |         | firewall policy is in use.           |
    +----------------+---------+--------------------------------------+

|

**Example Delete firewall policy: JSON request**

.. code::

    DELETE /v2.0/fw/firewall_policies/1be5e5f7-c45e-49ba-85da-156575b60d50.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Delete firewall policy: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0


Insert firewall rule in firewall policy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Inserts a firewall rule in a firewall policy relative to the position of
other rules.

    +----------------+------------------------------------------------+
    | Request Type   | ``PUT``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_policies/<firewall_policy_id>\  |
    |                | /insert_rule``                                 |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Not Found(404)    |
    |                |         | Bad Request(400) Bad Request error   |
    |                |         | response is returned when the rule   |
    |                |         | information is missing, Conflict(409)|
    +----------------+---------+--------------------------------------+

|

**Example Insert firewall rule in firewall policy: JSON request**

.. code::

    PUT /v2.0/fw/firewall_policies/41bfef97-af4e-4f6b-a5d3-4678859d2485/insert_rule.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule_id": "7bc34b8c-8d3b-4ada-a9c8-1f4c11c65692",
        "insert_after": "a08ef905-0ff6-4784-8374-175fffe7dade",
        "insert_before": ""
    }



**Example Insert firewall rule in firewall policy: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "audited": false,
        "description": "",
        "firewall_rules": [
            "a08ef905-0ff6-4784-8374-175fffe7dade",
            "7bc34b8c-8d3b-4ada-a9c8-1f4c11c65692",
            "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
        ],
        "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
        "name": "test-policy",
        "public": False,
        "project_id": "45977fa2dbd7482098dd68d0d8970117"
    }


Note:

insert_before and insert_after parameters refer to firewall rule uuids
already associated with the firewall policy. firewall_rule_id refers
to uuid of the rule being inserted.  When using "insert_after", if
there are any rules after the specified rule, they get shifted down by
one to later position. When using "insert_before", all rules from the
specified rule on get shifted down by one to a later position. Only
one of insert_after or insert_before can be non-null and if neither is
specified, firewall_rule_is inserted at the first position.

Remove firewall rule from firewall policy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Removes a firewall rule from a firewall policy.

    +----------------+------------------------------------------------+
    | Request Type   | ``PUT``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_policies/<firewall_policy_id>\  |
    |                | /remove_rule``                                 |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Not Found(404)    |
    |                |         | Bad Request(400) Bad Request error   |
    |                |         | response is returned when the rule   |
    |                |         | information is missing or when a     |
    |                |         | firewall rule is tried to be         |
    |                |         | removed from a firewall policy to    |
    |                |         | which it is not associated.          |
    +----------------+---------+--------------------------------------+

|

**Example Remove firewall rule from firewall policy: JSON request**

.. code::

    PUT /v2.0/fw/firewall_policies/41bfef97-af4e-4f6b-a5d3-4678859d2485/remove_rule.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule_id": "7bc34b8c-8d3b-4ada-a9c8-1f4c11c65692"
    }



**Example Remove firewall rule from firewall policy: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "audited": false,
        "description": "",
        "firewall_rules": [
            "a08ef905-0ff6-4784-8374-175fffe7dade",
            "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
        ],
        "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
        "name": "test-policy",
        "public": false,
        "project_id": "45977fa2dbd7482098dd68d0d8970117"
    }


List firewall groups
^^^^^^^^^^^^^^^^^^^^^

Lists firewall groups.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_groups``                        |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401)                    |
    +----------------+---------+--------------------------------------+

|

**Example List firewall groups: JSON request**

.. code::

    GET /v2.0/fw/firewall_groups.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example List firewall groups: JSON response**


.. code::

    {
        "firewall_groups": [
            {
                "description": "",
                "ingress_firewall_policy_id": null,
                "egress_firewall_policy_id": null,
                "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
                "name": "FW_GROUP_1",
                "project_id": "45977fa2dbd7482098dd68d0d8970117",
                "ports":[
                     "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
                 ],
            }
        ]
    }



Show firewall group details
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Shows firewall group details.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_groups/<firewall_group_id>``    |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Forbidden(403), \ |
    |                |         | Not Found (404)                      |
    +----------------+---------+--------------------------------------+

|

**Example Show firewall group: JSON request**

.. code::

    GET /v2.0/fw/firewall_groups/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json


**Example Show firewall group: JSON response**

.. code::

    {
        "firewall_group": {
            "description": "",
            "ingress_firewall_policy_id": null,
            "egress_firewall_policy_id": null,
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "FW_GROUP_1",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "ports":[
                 "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
             ],
        }
    }


Create firewall group
^^^^^^^^^^^^^^^^^^^^^^

Creates a firewall group.

    +----------------+------------------------------------------------+
    | Request Type   | ``POST``                                       |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_groups/``                       |
    +----------------+---------+--------------------------------------+
    |                | Success | 201                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400)  |
    +----------------+---------+--------------------------------------+

|

**Example Create firewall group: JSON request**

.. code::

    POST /v2.0/fw/firewall_groups.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule": {
            "ports":[
                 "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
             ],
            "name": "FW_GROUP_1"
        }
    }

**Example Create firewall group: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall_group": {
            "description": "",
            "ingress_firewall_policy_id": null,
            "egress_firewall_policy_id": null,
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "FW_GROUP_1",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "ports":[
                 "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
             ],
        }
    }


Update firewall group
^^^^^^^^^^^^^^^^^^^^^^^

Updates a firewall group.

    +----------------+------------------------------------------------+
    | Request Type   | ``PUT``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_groups/<firewall_group_id>``    |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400) \|
    |                |         | Not Found(404)                       |
    +----------------+---------+--------------------------------------+

|

**Example Update firewall group: JSON request**

.. code::

    PUT /v2.0/fw/firewall_groups/41bfef97-af4e-4f6b-a5d3-4678859d2485.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_group": {
            "ingress_firewall_policy_id": "f5876h5t1-b472-44f9-8245-308dr4u953gh2",
            "egress_firewall_policy_id": "dg476h5t1-b472-44f9-8245-308dr4u953gh2"
        }
    }



**Example Update firewall group: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall_group": {
            "description": "",
            "ingress_firewall_policy_id": "f5876h5t1-b472-44f9-8245-308dr4u953gh2",
            "egress_firewall_policy_id": "dg476h5t1-b472-44f9-8245-308dr4u953gh2",
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "FW_GROUP_1",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "ports":[
                 "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
             ],
        }
    }



Delete firewall group
^^^^^^^^^^^^^^^^^^^^^^^

Deletes a firewall group.

This operation does not return a response body.

    +----------------+------------------------------------------------+
    | Request Type   | ``DELETE``                                     |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/fw/firewall_groups/<firewall_group_id>``    |
    +----------------+---------+--------------------------------------+
    |                | Success | 204                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Not Found(404)    |
    +----------------+---------+--------------------------------------+

|

**Example Delete firewall group: JSON request**

.. code::

    DELETE /v2.0/fw/firewall_groups/1be5e5f7-c45e-49ba-85da-156575b60d50.json
    User-Agent: python-neutronclient
    Accept: application/json

**Example Delete firewall group: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0


Data Model Impact
------------------

The following are the backend database tables for the REST API proposed above.

|
| **Firewall Address Groups**


+-------------------+---------+-------+------+----------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                            |
+===================+=========+=======+======+========================================+
| id                | uuid-str| N/A   | R    | Unique identifier for the              |
|                   |         |       |      | address_group object.                  |
+-------------------+---------+-------+------+----------------------------------------+
| name              | String  | No    | CRU  | Human readable name for the address    |
|                   |         |       |      | group (255 characters limit). Does not |
|                   |         |       |      | have to be unique.                     |
+-------------------+---------+-------+------+----------------------------------------+
| description       | String  | No    | CRU  | Human readable description for the     |
|                   |         |       |      | address group (255 characters limit).  |
+-------------------+---------+-------+------+----------------------------------------+
| project_id        | uuid-str| Yes   | CR   | Owner of the address group. Only       |
|                   |         |       |      | admin users can specify a project      |
|                   |         |       |      | identifier other than their own.       |
+-------------------+---------+-------+------+----------------------------------------+


|
| **Firewall Address Group CIDR associations**

+-------------------+---------+-------+------+----------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                            |
+===================+=========+=======+======+========================================+
| id                | uuid-str| N/A   | R    | Unique identifier for the              |
|                   |         |       |      | address_group object.                  |
+-------------------+---------+-------+------+----------------------------------------+
| firewall_address  | uuid-str| No    | CRU  | UUID of firewall address group.        |
| _group_id         |         |       |      |                                        |
+-------------------+---------+-------+------+----------------------------------------+
| cidr              | String  | No    | CRU  | CIDR that has to be associated to the  |
|                   |         |       |      | firewall address group.                |
+-------------------+---------+-------+------+----------------------------------------+
| ip_version        | Integer | No    | CRU  | IP Protocol Version of the cidr.       |
+-------------------+---------+-------+------+----------------------------------------+



|
| **Firewall Rules**


+------------------------+------------+-----+------+---------------------------------------+
| Attribute              | Type       | Req | CRUD |  Description                          |
+========================+============+=====+======+=======================================+
| id                     | uuid-str   | N/A | R    | Unique identifier for the firewall    |
|                        |            |     |      | rule object.                          |
+------------------------+------------+-----+------+---------------------------------------+
| project_id             | uuid-str   | Yes | CRU  | Owner of the firewall rule. Only      |
|                        |            |     |      | admin users can specify a project     |
|                        |            |     |      | identifier other than their own.      |
+------------------------+------------+-----+------+---------------------------------------+
| name                   | String     | No  | CRU  | Human readable name for the firewall  |
|                        |            |     |      | rule (255 characters limit). Does     |
|                        |            |     |      | not have to be unique.                |
+------------------------+------------+-----+------+---------------------------------------+
| description            | String     | No  | CRU  | Human readable description for the    |
|                        |            |     |      | firewall Rule (255 characters limit). |
+------------------------+------------+-----+------+---------------------------------------+
| public                 | Bool       | No  | CRU  | When set to True makes this firewall  |
|                        |            |     |      | rule visible to projects other than   |
|                        |            |     |      | its owner, and can be used in         |
|                        |            |     |      | firewall policies not owned by its    |
|                        |            |     |      | project.                              |
+------------------------+------------+-----+------+---------------------------------------+
| protocol               | String     | No  | CRU  | IP Protocol.                          |
+------------------------+------------+-----+------+---------------------------------------+
| source_port            | port-range | No  | CRU  | Source port number or a range (an     |
|                        |            |     |      | int in [1, 65535] or range in a:b).   |
+------------------------+------------+-----+------+---------------------------------------+
| destination_port       | port-range | No  | CRU  | Destination port number or a range (  |
|                        |            |     |      | an int in [1, 65535] or range in a:b).|
+------------------------+------------+-----+------+---------------------------------------+
| service_group_id       | uuid-str   | No  | CRU  | UUID of the service group [6].        |
+------------------------+------------+-----+------+---------------------------------------+
| ip_version             | Integer    | No  | CRU  | IP Protocol Version.                  |
+------------------------+------------+-----+------+---------------------------------------+
| source_ip_address      | String     | No  | CRU  | Source IP address or CIDR.            |
+------------------------+------------+-----+------+---------------------------------------+
| destination_ip_address | String     | No  | CRU  | Destination IP address or CIDR.       |
+------------------------+------------+-----+------+---------------------------------------+
| source_address         | uuid-str   | No  | CRU  | When a source_address_group is        |
| _group_id              |            |     |      | specified, it is matched when the     |
|                        |            |     |      | source IP address in the packet       |
|                        |            |     |      | matches one of the IP addresses in    |
|                        |            |     |      | the address group.                    |
+------------------------+------------+-----+------+---------------------------------------+
| destination_address    | uuid-str   | No  | CRU  | When a destination_address_group is   |
| _group_id              |            |     |      | specified, it is matched when the     |
|                        |            |     |      | destination IP address in the packet  |
|                        |            |     |      | matches one of the IP addresses in the|
|                        |            |     |      | address group.                        |
+------------------------+------------+-----+------+---------------------------------------+
| source_firewall_group  | uuid-str   | No  | CRU  | When a source_firewall_group is       |
| _id                    |            |     |      | specified, it is matched when the     |
|                        |            |     |      | source IP address in the packet       |
|                        |            |     |      | matches an IP address of one of the   |
|                        |            |     |      | ports in the firewall group.          |
+------------------------+------------+-----+------+---------------------------------------+
| destination_firewall   | uuid-str   | No  | CRU  | When a destination_firewall_group is  |
| _group_id              |            |     |      | specified, it is matched when the     |
|                        |            |     |      | destination IP address in the packet  |
|                        |            |     |      | matches an IP address of one of the   |
|                        |            |     |      | ports in the firewall group.          |
+------------------------+------------+-----+------+---------------------------------------+
| action                 | String     | No  | CRU  | Action to be performed on the         |
|                        |            |     |      | traffic matching the rule (ALLOW,     |
|                        |            |     |      | DENY, REJECT). Default: DENY.         |
+------------------------+------------+-----+------+---------------------------------------+
| enabled                | Bool       | No  | CRU  | When set to False will disable this   |
|                        |            |     |      | rule in the firewall policy.          |
|                        |            |     |      | Facilitates selectively turning off   |
|                        |            |     |      | rules without having to disassociate  |
|                        |            |     |      | the rule from the firewall policy.    |
|                        |            |     |      | Default: True.                        |
+------------------------+------------+-----+------+---------------------------------------+


|
| **Firewall Policies**

+----------------+------------+-----+------+-----------------------------------------+
| Attribute      | Type       | Req | CRUD | Description                             |
+================+============+=====+======+=========================================+
| id             | uuid-str   | N/A | R    | Unique identifier for the firewall      |
|                |            |     |      | policy object.                          |
+----------------+------------+-----+------+-----------------------------------------+
| project_id     | uuid-str   | Yes | CR   | Owner of the firewall policy. Only      |
|                |            |     |      | admin users can specify a project       |
|                |            |     |      | identifier other than their own.        |
+----------------+------------+-----+------+-----------------------------------------+
| name           | String     | No  | CRU  | Human readable name for the firewall    |
|                |            |     |      | policy (255 characters limit). Does     |
|                |            |     |      | not have to be unique.                  |
+----------------+------------+-----+------+-----------------------------------------+
| description    | String     | No  | CRU  | Human readable description for the      |
|                |            |     |      | firewall Policy (255 characters limit). |
+----------------+------------+-----+------+-----------------------------------------+
| audited        | Bool       | No  | CRU  | When set to True by the policy owner    |
|                |            |     |      | indicates that the firewall policy has  |
|                |            |     |      | been audited. Each time the firewall    |
|                |            |     |      | policy or the associated firewall       |
|                |            |     |      | rules are changed, this attribute will  |
|                |            |     |      | be set to False and will have to be     |
|                |            |     |      | explicitly set to True through an       |
|                |            |     |      | update operation.                       |
+----------------+------------+-----+------+-----------------------------------------+
| public         | Bool       | No  | CRU  | When set to True makes this firewall    |
|                |            |     |      | policy visible to projects other than   |
|                |            |     |      | its owner.                              |
+----------------+------------+-----+------+-----------------------------------------+

|
| **Firewall Policy Rule associations**

+--------------------+---------+-------+------+---------------------------------------+
| Attribute          | Type    | Req   | CRUD | Description                           |
+====================+=========+=======+======+=======================================+
| id                 | uuid-str| N/A   | R    | Unique identifier for the             |
|                    |         |       |      | firewall_policy_rules object.         |
+--------------------+---------+-------+------+---------------------------------------+
| firewall_policy_id | uuid-str| No    | CRU  | UUID of the firewall policy.          |
+--------------------+---------+-------+------+---------------------------------------+
| firewall_rule_id   | uuid-str| No    | CRU  | UUID of the firewall rule.            |
+--------------------+---------+-------+------+---------------------------------------+
| position           | Integer | No    | CRU  | This is an attribute that             |
|                    |         |       |      | gets assigned to this rule when the   |
|                    |         |       |      | rule is associated with a firewall    |
|                    |         |       |      | policy. It indicates the position of  |
|                    |         |       |      | this rule in that firewall policy.    |
+--------------------+---------+-------+------+---------------------------------------+


|
| **Firewall Groups**

+-------------------+---------+-------+------+---------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                           |
+===================+=========+=======+======+=======================================+
| id                | uuid-str| N/A   | R    | Unique identifier for the firewall    |
|                   |         |       |      | group object.                         |
+-------------------+---------+-------+------+---------------------------------------+
| name              | string  | No    | CRU  | Human readable name for the firewall  |
|                   |         |       |      | group (255 characters limit). Does    |
|                   |         |       |      | not have to be unique.                |
+-------------------+---------+-------+------+---------------------------------------+
| description       | string  | No    | CRU  | Human readable description for the    |
|                   |         |       |      | firewall group (255 characters limit).|
+-------------------+---------+-------+------+---------------------------------------+
| project_id        | uuid-str| Yes   | CR   | Owner of the firewall group. Only     |
|                   |         |       |      | admin users can specify a project     |
|                   |         |       |      | identifier other than their own.      |
|                   |         |       |      | Default: derived from authentication  |
|                   |         |       |      | token.                                |
+-------------------+---------+-------+------+---------------------------------------+
| ingress_firewall  | uuid-str| No    | CRU  | 'null' if not associated with any     |
| _policy_id        |         |       |      | firewall policy.                      |
+-------------------+---------+-------+------+---------------------------------------+
| egress_firewall   | uuid-str| No    | CRU  | 'null' if not associated with any     |
| _policy_id        |         |       |      | firewall policy.                      |
+-------------------+---------+-------+------+---------------------------------------+


|
| **Firewall Group Port associations**

+-------------------+--------+-------+------+---------------------------------------+
| Attribute         | Type   | Req   | CRUD | Description                           |
+===================+========+=======+======+=======================================+
| id                | uuid   | N/A   | R    | Unique identifier for the             |
|                   |        |       |      | firewall_group_port object.           |
+-------------------+--------+-------+------+---------------------------------------+
| firewall_group_id | uuid   | No    | CRU  | UUID of the firewall group.           |
+-------------------+--------+-------+------+---------------------------------------+
| port_id           | uuid   | No    | CRU  | UUID of the port that will be         |
|                   |        |       |      | associated to this firewall_group.    |
|                   |        |       |      | It can be any Neutron port (VM ports, |
|                   |        |       |      | router ports, SFC ports).             |
+-------------------+--------+-------+------+---------------------------------------+

|
|

Multiple Firewall Policies
---------------------------

When only one firewall group is associated with a specific Neutron port,
the firewall rules are evaluated in order according to their position.
Both "allow" and "deny" rules can be interspersed in any order, with the
first match determining the action to be taken.

The FWaaS 2.0 API allows for multiple firewall group associations for
the same Neutron port. For example, one firewall group applied to a
tenant's web servers may specify a firewall policy that restricts
traffic to HTTP only (intended for north/south traffic), while another
firewall group applied to all the tenant's instances may specify a
firewall policy that allows all traffic types between sources and
destinations in that firewall group (east/west traffic).

When there are multiple firewall groups associated with a specific
Neutron port, there is no position or priority between the different
firewall groups. Some deterministic behavior must be defined in order to
resolve the action to be taken when some firewall groups determine an
"allow" action while other firewall groups determine a "deny" action.

This spec defines that packets will be allowed if any one of the
firewall groups associated with that Neutron port allows the packet.
This behavior is similar to the case of multiple Security Groups
associated with the same VM port.

In future phases, new constructs will be proposed that will allow for
for a "deny" action determined by one firewall group to override an
"allow" action determined by a different firewall group in some cases,
depending on how the firewall groups are associated with the Neutron
port. Refer "Stratum" section in [3] for further details.


Security Impact
---------------

* **TBD**

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

In case of DVR, router ports other than north-facing router ports will
not be supported. The asymmetric design of the DVR data plane for
east/west traffic prevents the use of connection tracking, which is
essential for proper FWaaS operation. The workaround in order to apply
FWaaS to east/west traffic is to enforce FWaaS at VM ports rather than
router ports. Enforcement of FWaaS at VM ports seems to be better
aligned with the distributed architecture of DVR, regardless of this
restriction.

Performance Impact
------------------

TBD.

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

* We will need new Heat models.

Developer Impact
----------------

None.

Community Impact
----------------

None.

Alternatives
------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
* Sean M. Collins

Other contributors:
* Aishwarya Thangappa
* German Eichberger
* James Ardent
* Mickey Spiegel
* Sridhar Kandaswamy

Work Items
----------

* REST API
* DB Schema
* CLI update

Dependencies
============

* Depends on the Service Groups [6]

Testing
=======

Tempest Tests
--------------

* DB mixin and schema tests
* FWaaS Plugin with mocked driver end-to-end tests
* Tempest tests
* CLI tests

Functional Tests
----------------

* New tests need to be written

API Tests
---------

* REST API and attributes validation tests

Documentation Impact
====================

User Documentation
-------------------

* Neutron CLI and FWaaS API documentation have to be modified.

Developer Documentation
-----------------------

* neutron-fwaas repo will have a devref and documentation will be written.

References
===========

[1] https://www.openstack.org/summit/tokyo-2015/videos/presentation/openstack-neutron-fwaas-roadmap

[2] https://etherpad.openstack.org/p/mitaka-neutron-next-adv-services

[3] https://etherpad.openstack.org/p/fwaas-api-evolution-spec

[4] https://etherpad.openstack.org/p/FWaaS_with_DVR

[5] https://bugs.launchpad.net/neutron/+bug/1513574

[6] http://specs.openstack.org/openstack/neutron-specs/specs/kilo/service-group.html

[7] http://developer.openstack.org/api-ref-networking-v2-ext.html#security_groups

