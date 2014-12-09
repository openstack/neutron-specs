============
Extra routes
============

This extension adds extra routes to the ``router`` resource.

You can specify a set of nexthop IPs and destination CIDRs.

Note
~~~~

The nexthop IP must be a part of one of the subnets to which the router
interfaces are connected. You can configure the ``routes`` attribute on
only update operations.

**Table Router attributes**

Attribute

Type

Required

CRUD\ `:sup:`[a]` <#ftn.crud_extraroute>`__

Default Value

Validation Constraints

Notes

routes

list of dict

No

U

None

List should be in this form. [{'nexthop':IPAddress, 'destination':CIDR}]

Extra route configuration

-  **`:sup:`[a]` <#crud_extraroute>`__\ C**. Use the attribute in create
   operations.

-  **R**. This attribute is returned in response to show and list
   operations.

-  **U**. You can update the value of this attribute.

-  **D**. You can delete the value of this attribute.



Update extra route
~~~~~~~~~~~~~~~~~~

**PUT**

/routers/*``router_id``*

Updates logical router with ``routes`` attribute.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Bad Request (400), Not Found
(404), Conflict (409)

This operation configures extra routes on the router. The nexthop IP
must be a part of one of the subnets to which the router interfaces are
connected. Otherwise, the server responds with ``400 Bad Request`` error
code. When a validation error is detected, such as a format error of IP
address or CIDR, the server responds with ``400 Bad Request``. When
Networking receives a request to delete the router interface for subnets
that are used by one or more routes, it responds with ``409 Conflict``.

**Example Update routes: JSON request**

.. code::

    {
       "router":{
          "routes":[
             {
                "nexthop":"10.1.0.10",
                "destination":"40.0.1.0/24"
             }
          ]
       }
    }



**Example Update routes: JSON response**

.. code::

    {"router":
        {"status": "ACTIVE",
         "external_gateway_info": {"network_id": "5c26e0bb-a9a9-429c-9703-5c417a221096"},
         "name": "router1",
         "admin_state_up": true,
         "tenant_id": "936fa220b2c24a87af51026439af7a3e",
         "routes": [{"nexthop": "10.1.0.10", "destination": "40.0.1.0/24"}],
         "id": "babc8173-46f6-4b6f-8b95-38c1683a4e22"}
    }



