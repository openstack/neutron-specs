..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Neutron Common Classification Framework
=======================================

https://blueprints.launchpad.net/neutron/+spec/common-classification-framework
https://wiki.openstack.org/wiki/Neutron/CommonClassificationFramework

Problem Description
===================

Different Neutron projects, services, features and related projects have
different traffic classifiers or classification APIs.
They differ on both syntax and semantics. A few examples are:

- Security group rules [6]
- openstack/neutron-fwaas [7]
- openstack/networking-sfc (Flow Classifier) [8]
- openstack/neutron-classifier [2]
- openstack/networking-bgpvpn [12]
- openstack/tap-as-a-service [13]
- Neutron QoS [10]

Furthermore, there are other projects with classification APIs, such as
openstack/group-based-policy [9] and it's possible that more will want
to support classifications in their own APIs, further reinventing the wheel
and fragmenting the language used in the OpenStack ecosystem when it comes
to defining traffic classifications.

This spec introduces the Neutron Common Classification Framework (CCF),
a culmination of previous efforts: Common Classifier [0],
openstack/neutron-classifier [2] and Common Flow Classifier [11].

The previous spec for this RFE is archived at [0].


Proposed Change
===============

Terminology:

- **Common Classification Framework (CCF)**: the name of this project, which
  includes the base models and resources introduced below, the API, logic and
  versioned objects to allow Consuming Services to consume from the framework.
- **Classification**: the Neutron resource being introduced for the purpose of
  defining an atomic traffic classification.
- **Classification Group (CG)**: the Neutron resource being introduced for the
  purpose of grouping Classifications, or other Classification Groups,
  together, specifying a common relation across them. This is the resource
  that ultimately will be consumed by a Consuming Service.
- **Classification Type (CT)**: every Classification has a type, defining the
  format, supported fields and syntax validators of that classification.
- **Classifications API**: the User-facing REST API that is provided by the
  Common Classification Framework to instantiate the resources above.
- **Consuming Service (CS)**: Neutron itself or a Neutron project, service,
  extension or feature that is capable of consuming Classification Groups
  (acquiring them as resources), examples of potential Consuming Services are
  the ones listed in Problem Description.
  These services do not talk directly to the Classifications API.
- **User**: the end-user, human or machine, tenant or admin, typically cloud
  consumer, calling into the Classifications API and any Consuming Services'
  REST APIs.

This spec defines the new kinds of resources to be introduced, an initial set
of Classification Types and their models, and the Classifications API in order
to define such traffic classifications, which the User will call.

The dedicated Classifications API provides a way to define and create
Classifications and CGs, storing them like any other resource. CSs are
then able to fetch those traffic classification definitions as
oslo.versionedobjects (OVO) and feed them to their respective internal network
classifiers (i.e. to the mechanism capable of matching on traffic, such as
Open vSwitch via the respective ML2 mechanism driver, or
iptables - it really depends on what service, and its configuration, is
consuming the Classification Groups). What Consuming Services do with the
resources consumed is entirely out of scope from the CCF.

Updating Classification or CG resources is disallowed in the API, except for
changes not related to the traffic classification semantics, such as name and
description. With this, having to deal with the complexities of advertising
resource changes to CSs is also avoided which, at least for a first version
of the CCF, is beneficial as it keeps the project simpler.

Consuming Services are not required to keep a local
database copy of the consumed classifications, beyond mapping CG UUIDs with
Consuming Services' own resources. Consequently, the Common
Classification Framework should prevent Classifications from being deleted if
they are in use by at least 1 resource of any Consuming Service.

Summary of changes necessary to create the Common Classification Framework:

- Creation of a Neutron extension to expose the new, first version of the
  Classifications API, as additional modifications will require new extensions.
- Creation of models for the different supported protocols' types/definitions.
- Creation of a Neutron service plugin and DB layer to validate
  the Classifications defined by the API according to the types defined and
  respective supported models, and store them in the database.
- Adoption of OVO so that every Classification defined becomes a versioned and
  serializable object, which will be transferred from the Common Classification
  Framework to Consuming Services.
- Create CLI using OSC so Users can define their own Classifications with ease.

Though out of scope, this is a summary of changes needed in Consuming Services:

- Add a dependency on the CCF's own extension.
- Add (or reuse) fields to/in certain API resources with the intent of
  receiving Classification Group UUIDs (e.g. networking-sfc port-chain's
  flow_classifiers field could expect a common Classification Group UUID
  instead of local Flow Classifier UUIDs).
- Evaluate the Classifications provided to ascertain whether they are
  compatible with the specific extension's classifier (mechanism).
- Add logic to fetch the CGs provided by UUID, making
  use of OVO in the process, potentially via neutron-lib (xgerman).
- Translate the classifications' definitions to the specific classifier's
  mechanism or language (e.g. create Open vSwitch flows matching on traffic).
- Typically Consuming Services will already have their own CLI, so changes
  must be made consistently with the changes made to their own APIs and/or
  classification-fetching code.

The diagram at [5] shows the relationship between Users, the Common
Common Classification Framework and the Consuming Services.


Data Model Impact
-----------------

Grouping traffic classifications by individual types, called Classification
Types, will allow future types to be added and agreed in the future, while
keeping the remaining ones intact. Existing types can of course be updated,
thanks to versioning. Thus, this approach follows a modular, rather than
a monolithic way of defining classifications. Looking at the existing
neutron-classifier project [2], which was decided as the repo to keep the
Common Classifier [3] (during the Tokyo Summit 2015), there are hints of an
architecture like the one proposed here, as can be seen in [4]. As such, this
framework is partially based on the neutron-classifier with the major
difference that it presents a REST API for Users to define classifications.

Before delving into further detail in regards to the data model, API and how
classifications can be used by interested Neutron subprojects, a few points
need to be clarified:

- Classification Types can be introduced or extended (with new fields e.g.) in
  every release of the CCF. API extensions will be added to reflect these
  additions in the REST API and maintain backwards compatibility.

- 1 Classification is of a single type, e.g. either Ethernet, IP, HTTP,
  or another supported at the time of a specific CCF release. The definition,
  i.e. fields to match on, depends on the type specified.

- To clarify, Classification Types define the set of possible fields and values
  for a Classification (essentially, an instance of that Classification Type).
  Classification Types are defined in code, where Classifications are created
  via the REST API as instances of those types.

- Not all supported fields need to be defined - only the ones
  required by the Consuming Service - which it should validate on consumption.

- There are also Classification Groups, which allow Classifications or other
  Classification Groups to be grouped together using boolean operators. CGs
  are the resources that will end up being consumed by Consuming Services.

- The CCF has to be able to check if a Classification Group
  is currently being used, and prevent it from getting deleted if so.

- From the Consuming Service's point of view, Classifications can only be read,
  not created or deleted. They need to have been previously
  created using the User-facing Classifications API.
  Figure [5] attempts to illustrate this.

The initial model of the CCF will includes the following Classification Types:
Ethernet, IPv4, IPv6, TCP and UDP, which when combined are sufficient
to provide any 5-tuple classification.


The following table presents the attributes of a Classification Group
(asterisk on RW means that the attribute is non-updatable):

 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | Attribute            | Type    | Access | Default   | Validation/ | Description                 |
 | Name                 |         | CRUD   | Value     | Conversion  |                             |
 +======================+=========+========+===========+=============+=============================+
 | id                   | string  | RO,    | generated | uuid        | Identity                    |
 |                      | (UUID)  | all    |           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | project_id           | string  | RO,    | from auth | uuid        | Project ID                  |
 |                      | (UUID)  | project| token     |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | name                 | string  | RW,    | None      | string      | Name of Classification Group|
 |                      |         | project|           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | description          | string  | RW,    | None      | string      | Human-readable description  |
 |                      |         | project|           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | shared               | bool    | RW,    | False     | boolean     | Shared with other projects  |
 |                      |         | project|           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | operator             | string  | RW*,   | "and"     | ["and",     | Boolean connective: AND/OR  |
 |                      | (values)| project|           |  "or"]      |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | classification_groups| list    | RW*,   | []        |             | List of Classification      |
 |                      |         | project|           |             | Groups included             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | classifications      | list    | RW*    | []        |             | List of Classifications     |
 |                      |         | project|           |             | included                    |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+

Consuming Services will consume Classification Groups, and not atomic
Classifications (that would create more difficulties in terms of the
relationships between CCF and CSs databases), any Classification needs to
be grouped in a Classification Group to be consumed individually. As such,
the "operator" field is to be ignored for Classification Groups that only
contain 1 Classification inside.

The following table presents the attributes of Classifications
of any of the types stated in this spec
(asterisk on RW means that the attribute is non-updatable):

 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | Attribute            | Type    | Access | Default   | Validation/ | Description                 |
 | Name                 |         |        | Value     | Conversion  |                             |
 +======================+=========+========+===========+=============+=============================+
 | id                   | string  | RO,    | generated | uuid        | Identity                    |
 |                      | (UUID)  | all    |           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | project_id           | string  | RO,    | from auth | uuid        | Project ID                  |
 |                      | (UUID)  | project| token     |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | name                 | string  | RW,    | None      | string      | Name of Classification      |
 |                      |         | project|           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | description          | string  | RW,    | None      | string      | Human-readable description  |
 |                      |         | project|           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | type                 | string  | RW*,   |           | from enum   | The type of the             |
 |                      |         | project|           | of types    | Classification              |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | negated              | bool    | RW*,   | False     | boolean     | Whether to negate           |
 |                      |         | project|           |             | classification (boolean NOT)|
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | definition           | type-specific attributes will go here,                                   |
 |                      | given their volume I won't detail them unless requested.                 |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+


Classification Groups and Classifications of every type will be stored as the
following tables and relationships (with table name prefix ``ccf_``)::

                           +---------------------+
                           |classification_groups|
                           +---------------------+
                           |id                   |*
                           |cg_id                +--------+
                           |name                 |        |
                           |description          |        |
                           |project_id           |        |
                           |shared               +--------+
                           |operator             |1
                           +---------------------+
                                       |1
                                       |
                                       |*
                       +------------------------------+
                       |classification_groups_mapping |
                       +------------------------------+
                       |cg_id                         |
                       |classification_id             |
                       +------------------------------+
                                       |1
 +--------------------+                |                +--------------------+
 |ipv4_classifications|                |                |ipv6_classifications|
 +--------------------+                |                +--------------------+
 |classification_id   |                |                |classification_id   |
 |ihl                 |1               |               1|traffic_class       |
 |diffserv            +--------+       |       +--------+traffic_class_mask  |
 |diffserv_mask       |        |       |       |        |length              |
 |length              |        |       |       |        |next_header         |
 |flags               |        |       |       |        |hops                |
 |flags_mask          |        |       |       |        |src_addr            |
 |ttl                 |        |1      |1     1|        |dst_addr            |
 |protocol            |     +---------------------+     +--------------------+
 |src_addr            |     |classifications      |
 |dst_addr            |     +---------------------+
 |options             |     |id                   |
 |options_mask        |     |name                 |
 +--------------------+     |description          |
                            |project_id           |
                            |shared               |     +-------------------+
                            |type                 |     |tcp_classifications|
                            |negated              |     +-------------------+
                            +---------------------+     |classification_id  |
 +-------------------+        1|      1|       |1       |src_port           |
 |udp_classifications|         |       |       |        |dst_port           |
 +-------------------+         |       |       |        |flags              |
 |classification_id  |1        |       |       |       1|flags_mask         |
 |src_port           +---------+       |       +--------+window             |
 |dst_port           |                1|                |data_offset        |
 |length             |     +------------------------+   |option_kind        |
 |window_size        |     |ethernet_classifications|   +-------------------+
 +-------------------+     +------------------------+
                           |classification_id       |
                           |preamble                |
                           |src_addr                |
                           |dst_addr                |
                           |ethertype               |
                           +------------------------+


Some of the fields of the Classification Types presented above in the database
schema, such as ``length``, ``src_addr``, and others, will allow ranges or
lists to be input, through the use of commas or hyphens, for example.

Masking fields allow the user to specify which individual bits of the
respective main field should be looked up during classification.

Besides the Classification Types presented above, the following types are also
expected to be part of the first release of the CCF:
- Neutron (destination/source port, subnets, networks, at least)
- ICMP
- ICMPv6
- SCTP
- ARP
- VLAN
- GRE
- VXLAN
- Geneve
- MPLS
- NSH

Classification Types are used to select the appropriate model of the
Classification and consequently what table it will be stored in.

Classification Groups get stored in a single table and can point to other
Classification Groups, to allow mixing boolean operators.

There are two important fields meant for boolean logic:

- ``operator`` in Classification Group: specifies the boolean operator used
  to connect all the child Classifications and Classification Groups of that
  group. This can be either AND or OR.

- ``negated`` per Classification "usage": specifies whether to negate the
  definition of the Classification, when mapped to a Classification Group,
  essentially a boolean NOT. This can be True or False. Please note that
  Classification Groups cannot be negated using this model.


REST APIs
---------

A new API extension is being introduced. The base URL
for the Classifications API is /v2.0/.

The following table summarizes available URIs::

 +----------------------+------------------------------------+-------+
 |Resource              |URI                                 |Type   |
 +======================+====================================+=======+
 |classification_types  |/classification_types               |GET    |
 +----------------------+------------------------------------+-------+
 |classification_group  |/classification_groups/             |POST   |
 +----------------------+------------------------------------+-------+
 |classification_groups |/classification_groups              |GET    |
 +----------------------+------------------------------------+-------+
 |classification_group  |/classification_groups/{id}         |GET    |
 +----------------------+------------------------------------+-------+
 |classification_group  |/classification_groups/{id}         |PUT    |
 +----------------------+------------------------------------+-------+
 |classification_group  |/classification_groups/{id}         |DELETE |
 +----------------------+------------------------------------+-------+
 |classification        |/classifications/                   |POST   |
 +----------------------+------------------------------------+-------+
 |classifications       |/classifications                    |GET    |
 +----------------------+------------------------------------+-------+
 |classification        |/classifications/{id}               |GET    |
 +----------------------+------------------------------------+-------+
 |classification        |/classifications/{id}               |PUT    |
 +----------------------+------------------------------------+-------+
 |classification        |/classifications/{id}               |DELETE |
 +----------------------+------------------------------------+-------+

The CCF should provide a way to mark a Classification Group as being in use
(or increase the usage count) and a way to check for that and abort
certain operations if the group is in use.

The CCF does not provide any mechanism to synchronize Classification Groups
to Consuming Services.

Examples for a Classification Group with two Classifications inside.


To list available Classification Types::

  GET /v2.0/classification_types

  Response:
  {
     "classification_types": [{"type": "ethernet"},
                              {"type": "ipv4"},
                              {"type": "ipv6"},
                              {"type": "tcp"},
                              {"type": "udp"}]
  }


To create a Classification of type TCP::

  POST /v2.0/classifications/
  {
      "classification": {
          "name": "not_tcp_syns",
          "type": "tcp",
          "negated": true,
          "definition": {
              "control_flags": "0x2",
              "control_flags_mask: "0x2"
          }
      }
  }

  Response:
  {
      "classification": {
          "id": "3dcc561a-1bb8-11e7-b615-23717626a4e5",
          "project_id": "0a36035e-1bb9-11e7-b8ef-e782361fd276",
          "name": "not_tcp_syns",
          "description": "",
          "type": "tcp",
          "negated": true,
          "shared": false,
          "definition": {
              "src_port": null,
              "dst_port": null,
              "control_flags": "0x2",
              "control_flags_mask: "0x2",
              "ecn": null,
              "ecn_mask": null,
              "min_window": null,
              "max_window": null,
              "min_data_offset": null,
              "max_data_offset": null,
              "option_kind": null
          }
      }
  }


To create a Classification of type Ethernet::

  POST /v2.0/classifications/
  {
      "classification": {
          "name": "ipv4_over_eth",
          "type": "ethernet",
          "definition": {
              "ethertype": "0x800"
          }
      }
  }

  Response:
  {
      "classification": {
          "id": "021c1ad2-1bb9-11e7-907d-937a75c8a5db",
          "project_id": "0a36035e-1bb9-11e7-b8ef-e782361fd276",
          "name": "ipv4_over_eth",
          "description": "",
          "type": "ethernet",
          "negated": false,
          "shared": false,
          "definition": {
              "negated": false,
              "preamble": null,
              "src_addr": null,
              "dst_addr": null,
              "ethertype": "0x800"
          }
      }
  }


To create a Classification Group::

  POST /v2.0/classification_groups/
  {
      "classification_group": {
          "name": "no_syns_on_ipv4",
          "description": "Any IPv4 traffic carried over Ethernet except TCP SYNs.",
          "operator": "and",
          "classifications": [
              "3dcc561a-1bb8-11e7-b615-23717626a4e5",
              "021c1ad2-1bb9-11e7-907d-937a75c8a5db"
          ]
      }
  }

  Response:
  {
      "classification_group": {
          "id": "387299fa-250d-11e7-8620-b38d21865984",
          "project_id": "0a36035e-1bb9-11e7-b8ef-e782361fd276",
          "name": "no_syns_on_ipv4",
          "description": "Any IPv4 traffic carried over Ethernet except TCP SYNs.",
          "shared": false,
          "operator": "and"
          "classifications": [
              "3dcc561a-1bb8-11e7-b615-23717626a4e5",
              "021c1ad2-1bb9-11e7-907d-937a75c8a5db"
          ],
          "classification_groups": []
      }
  }


List Classification Groups::

 GET /v2.0/classification_groups

 Response:
 {
     "classification_groups": [
         {
             "id": "387299fa-250d-11e7-8620-b38d21865984",
             "project_id": "0a36035e-1bb9-11e7-b8ef-e782361fd276",
             "name": "no_syns_on_ipv4",
             "description": "Any IPv4 traffic carried over Ethernet except TCP SYNs.",
             "shared": false,
             "operator": "and"
             "classifications": [
                 "3dcc561a-1bb8-11e7-b615-23717626a4e5",
                 "021c1ad2-1bb9-11e7-907d-937a75c8a5db"
             ],
             "classification_groups": []
         },
         {
             ...
         }
     ]
 }


Community Impact
----------------

Services that intend to consume Classifications only have to:

- Modify their existing REST APIs slightly in order to allow for
  Classification Group UUIDs to be passed, if they don't already
  have an endpoint for such (e.g. networking-sfc doesn't require
  port-chain API resource changes).

- Implement the underlying fetching of Classification definitions,
  with support for OVO.


Other End User Impact
---------------------

The OpenStack Client will be extended to support the Classifications API.

Possible CLI syntax of a hypothetical scenario with Neutron QoS as the
Consuming Service and the Classification Group presented above in the
example API calls, to illustrate the workflow::

 $ openstack network classification create --type=tcp --control_flags=0x2 --control_flags_mask=0x2 --negated tcp_syns
 $ openstack network classification create --type=ethernet --ethertype=0x800 ipv4_over_eth
 $ openstack network classification group create --description "Any IPv4 traffic carried over Ethernet except TCP SYNs." \
       --classification tcp_syns --classification ipv4_over_eth --operator and no_syns_on_ipv4
 $ openstack network qos rule create --type bandwidth-limit --max-kbps 8000 --classification-group no_syns_on_ipv4 myqospolicy


Alternatives
------------

Possible alternatives to the data model and REST API presented are:

- Neutron QoS -inspired model;

- Using oslo.versionedobjects serialized into JSON for the REST API,
  to support the exact same resources specified in here.

- With regards to the initial model, it could as well include
  Security Group Rule UUIDs, BGP VPN resource UUIDs, HTTP, and many more.


Implementation
==============

Work has started as an initial Proof of Concept, available at [14].
After an initial merge on the neutron-classifier repository, work will
continue towards the goals outlined in this spec.


Assignee(s)
-----------

 - Igor Duarte Cardoso (igor.duarte.cardoso@intel.com)
 - David Shaughnessy (david.shaughnessy@intel.com)

We request the Neutron team to provide access to the neutron-classifier repo.

Given the findings with the PoC code, there is no expectation that existing
code in neutron-classifier will be reused, so the repository is to be wiped
(but all the history will be kept).


Work Items
----------

- Prototype/PoC (Pike-1) (David) - Finished.
- Finish spec (Pike) (Igor) - In Progress.
- Acquire rights to merge on a repository (Pike) (Igor) - In Progress.
- Implement first consumable version of this project (Pike to Queens) (David, Igor)
- Bring first time support to a Neutron service (TBD) (Queens to Rocky)


References
==========

.. [0] Add common classifier resource (neutron-specs): https://review.openstack.org/#/c/190463/
.. [1] Data Model: http://i.imgur.com/MPuOAvv.png
.. [2] The neutron-classifier project: http://git.openstack.org/cgit/openstack/neutron-classifier
.. [3] The original and current RFE to bring a common classifier to Neutron: https://bugs.launchpad.net/neutron/+bug/1476527
.. [4] neutron-classifier inspiration: https://github.com/openstack/neutron-classifier/blob/10b2eb3127f4809e52e3cf1627c34228bca80101/neutron_classifier/common/constants.py#L17
.. [5] Relationship between communicating entities: http://i.imgur.com/9jttN11.png
.. [6] Security group rules: http://developer.openstack.org/api-ref/networking/v2/?expanded=show-security-group-rule-detail
.. [7] FWaaS API spec: http://specs.openstack.org/openstack/neutron-specs/specs/api/firewall_as_a_service__fwaas_.html
.. [8] networking-sfc API: http://docs.openstack.org/developer/networking-sfc/api.html
.. [9] Group-based Policy: http://gbp.readthedocs.io/en/latest/usage.html
.. [10] Traffic classification in Neutron QoS: https://bugs.launchpad.net/neutron/+bug/1527671
.. [11] Common Flow Classifier proposed model: https://wiki.openstack.org/w/images/c/c8/neutron_common_classifier.png
.. [12] openstack/networking-bgpvpn API reference: https://docs.openstack.org/developer/networking-bgpvpn/api.html#bgpvpn-resource
.. [13] openstack/tap-as-a-service API reference: https://github.com/openstack/tap-as-a-service/blob/master/API_REFERENCE.rst
.. [14] Latest CCF PoC: https://review.openstack.org/#/c/445577/
