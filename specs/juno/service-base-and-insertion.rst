
==========================================
ServiceBase and Service Insertion
==========================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/service-base-class-and-insertion

This blueprint proposes to extend the Service Base class to capture common
service attributes and an extension to the Neutron API to facilitate service
insertion.

Problem description
===================

In Havana and IceHouse, Neutron advanced services have made significant progress.
LBaaS has been accepted as an extension, and FWaaS and VPNaas have been included
in Neutron as experimental extensions.

However, there isn't a common mechanism to specify service insertion criteria
across the existing services. For instance there isn't a way to specify that
a firewall has to be inserted on the edge of a particular network, or a
different firewall filters traffic only between two specific networks.
This blueprint extends the existing Service Base definition with common
properties and APIs to ensure the uniform service insertion semantics across
all Neutron services.


Proposed change
===============

A Neutron service instance is a logical entity that applies some network
functions onto streams of packets. E.g. a Firewall.

We propose inserting this service instance in the context of a Neutron port (or
its variation like the external-port [2]) or attaching to another Neutron
service instance.

Service Interface is a new resource that provides a place to hold this
insertion context information. Every time a service instance has to be
inserted to a neutron network, a new Service Interface will be created. Each
service can thus have zero or more Service Interfaces.

When a tenant requests creation of a logical service instance using
the Flavor id (the Flavor reference is mentioned here for completeness,
but there is no immediate dependency on it), a backend is selected to realize
the service instance. Subsequently, a request to insert the service instance
will result in the Service Interface information being propagated to the
backend which will realize the service based on the Service Interface context
information.

Example CLI:

Create Service Interface

::

 neutron create-service-interface <uuid_of_service_to_be_inserted>
 --insertion-type <neutron-port | external-port | service>
 <uuid_of_insertion_point>

Here the insertion point can be the uuid of a Neutron port, or Neutron External
Port or another Neutron service.

Delete Service Interface:

::

 neutron delete-service-interface <service_interface_uuid>


Alternatives
------------

An alternative is using serviceContext. The proposal is captured in

  - https://docs.google.com/document/d/
            1fmCWpCxAN4g5txmCJVmBDt02GYew2kvyRsh0Wl3YF2U/edit

and the corresponding patch:

  - https://review.openstack.org/#/c/62599/"

The alternative model intends to define the service insertion in a single
resource, serviceContext, which is consumed by the service at its instantiation.
However, the approach overlooked the fact that a service context is usually
constructed stepwise over time.
Furthermore, the alternative model permits ambiguous serviceContexts that
leads different interpretation in the backend implementation.

Data model impact
-----------------

* ServiceBase Class Diagram

::

                 +----------------+
                 |                |
                 |   ServiceBase  |
                 |                |
                 +------^---------+
                        |
     +--------------------------------------+
     |                  |                   |
 +----+------+      +----+------+     +------+-----+
 |           |      |           |     |            |
 |  FWaaS    |      |   LBaaS   |     |   VPNaaS   |
 |           |      |           |     |            |
 +-----------+      +-----------+     +------------+

  * New abstract methods in ServiceBase

    * get_supported_insertion_type()
      --- returns the type of interface context (neutron port, external
          port, or attach to service) supported, should only be one

    * create_service_interface(service_interface)
      --- create service interface

    * delete_service_interface(service_interface_id)
      --- delete service interface

    * get_service_interfaces()
      --- get the list of service interfaces for this service


* New Database Objects

::

 ServiceBase
  All Neutron services must inherit from this table class.

  * id            - uuid of a service
  * name          - optional name
  * description   - optional annotation
  * flavor_id     - uuid of the corresponding user requested flavor
  * service_interfaces - list of Service Interface uuids

 ServiceInterface

  * id - standard object uuid
  * name - optional name
  * description - optional annotation
  * tenant_id   - tenant who creates the service instance
  * insertion_type - enum: NEUTRON_PORT, EXTERNAL_PORT, SERVICE
  * insertion_point_id       - uuid of the insertion point, which can be one of
                               the following:
                               - uuid of a neutron port, if type == NEUTRON_PORT
                               - uuid of an external port, if type == EXTERNAL_PORT
                               - uuid of a service attached to this attachment point
                                 if type == SERVICE

Database Change:

* VPN DB change
  Remove router, name, and description attributes from VPN's DB table.
  Add reference to the corresponding ServiceBase entry

* LB DB change
  Remove port, name, and description attributes from LB's DB table.
  Add reference to the corresponding ServiceBase entry

* FW DB change
  Remove name and description attributes from FW's DB table.
  Add reference to the corresponding ServiceBase entry

Database migrations:

* VPN DB migration
  VPN currently has a router attribute in its DB to express the insertion.
  This proposal will remove the router attribute and create a Service
  Interface for each instance of VPN service. The Service Interface will
  have type=SERVICE and insertion_point_id=VPN's router value.

* Firewall DB migration
  The current firewall implementation inserts logical firewall on every
  router. This proposal enables the insertion of firewall to specific routers.
  In order to preserve the old behavior, during the database migration,
  one Service Interface would be created on the firewall for each router.
  The current firewall insertion is a subset of insertions supported in this
  proposal. Downgrade will not be possible.

* LB DB migration
  LB's VIP table has a port attribute to point to the Neutron port where
  the VIP is instantiated. This proposal will remove the port from VIP's DB
  and create a Service Interface for each instance of LB service. The
  Service Interface will have type=NEUTRON_PORT and ap_id is that of
  VIP's port.


REST API impact
---------------

The following new resources are being introduced:

.. code-block:: python

  sp_type = [None, 'NEUTRON_PORT', 'EXTERNAL_PORT', 'SERVICE']

  SERVICE_ATTACHMENT_POINTS = 'service_attachment_points'

  RESOURCE_ATTRIBUTE_MAP = {
      SERVICE_INTERFACES: {
          'id': {'allow_post': False, 'allow_put': False,
                 'validate': {'type:uuid': None}, 'is_visible': True,
                 'primary_key': True},
          'name': {'allow_post': True, 'allow_put': True,
                   'validate': {'type:string': None},
                   'default': '', 'is_visible': True},
          'description': {'allow_post': True, 'allow_put': True,
                          'validate': {'type:string': None},
                          'is_visible': True, 'default': ''},
          'service_id': {'allow_post': True, 'allow_put': False,
                         'validate': {'type:string': None},
                         'required_by_policy': True, 'is_visible': True},
          'insertion_type': {'allow_post': False, 'allow_put': False,
                             'validate': {'type:string': sp_type},
                             'default': None, 'is_visible': True},
          'insertion_point_id': {'allow_post': True, 'allow_put': True,
                                 'validate': {'type:uuid_or_none': None},
                                 'default': None, 'is_visible': True},
      },
  }

* Add "service_interfaces" attribute to the logical service instance resource
  for each service.

* Remove "router_id" from RESOURCE_ATTRIBUTE_MAP in extensions/vpnaas.py

* Remove "port_id" from RESOURCE_ATTRIBUTE_MAP in extensions/loadbalancer.py

The following is the default policy for service_context and service_interface:

.. code-block:: javascript

    {
        "create_service": "rule:admin_or_owner",
        "update_service": "rule:admin_or_owner",
        "get_service": "rule:admin_or_owner",
        "delete_service": "rule:admin_or_owner",
        "get_supported_insertion_type" : "rule:admin_or_owner"

        "create_service_interface": "rule:admin_or_owner",
        "get_service_interfaces": "rule:admin_or_owner",
        "delete_service_interface": "rule:admin_or_owner",
    }

Security impact
---------------

* Does this change touch sensitive data such as tokens, keys, or user data?

  No

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?

  No

* Does this change involve cryptography or hashing?

  No

* Does this change require the use of sudo or any elevated privileges?

  No

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.

  No

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

  No


Notifications impact
--------------------

None

Other end user impact
---------------------

Integration with following projects will be required:

* python-neutronclient
* horizon
* heat
* devstack

Performance Impact
------------------

The new APIs will be called when a service instance is inserted into the neutron
network. All performance considerations that are relevant to existing Neutron will
apply and be taken into consideration during the implementation. There is no major
change to the calling pattern of existing code.

Other deployer impact
---------------------

The existing FW, LB, and VPN databases will be migrated to the new table.
There should not be any impact. The default insertion of the existing
Neutron services will be supported as default.


Developer impact
----------------

VPN and LB APIs will need to take these changes into consideration for their
future revisions.
For backward compatibility, the current API will stay the same, but will be
marked for deprecation.
Any parallel efforts in XaaS, such as Tap-as-a-Service, should take this
framework into consideration from the beginning.


Implementation
==============

Assignee(s)
-----------

  Kanzhe Jiang (kanzhe-jiang)

  Kevin Benton (kevinbenton)

  Stephen Wong (s3wong)

  Sumit Naiksatam (snaiksat)

  Louis Fourie (louisf)

  Marios Andreou (marios)

  Sridar Kandaswamy (SridarK)

Work Items
----------

  Service Base changes
  Service Interface resource
  FWaaS, LBaaS, and VPNaaS updates to use this framework

Dependencies
============

This blueprint is under the umbrella of blueprint
neutron-services-insertion-chaining-steering [2].

This blueprint also depends on
neutron-external-ports [1]

Testing
=======

Both, functional and, system tests will be added.

Documentation Impact
====================

Both, API and, Admin guide will be updated.

References
==========

.. [1] External port
   https://review.openstack.org/#/c/87825

.. [2] https://review.openstack.org/92200
