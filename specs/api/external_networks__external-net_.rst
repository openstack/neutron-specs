================================
External networks (external-net)
================================

The external network extension is used to specify whether the network is
external or not. This information is used by Layer-3 network
(``router``) extension. External networks are connected to a router's
external gateway and host floating IPs.

The external network extension adds the router:external attribute to the
network resource.

CRUD\ `:sup:`[a]` <#ftn.crud_ext_net>`__

router:external type:Bool Not required, Default: False

Specifies whether the network is an external network or not.

-  **`:sup:`[a]` <#crud_ext_net>`__\ C**. Use the attribute in create
   operations.

-  **R**. This attribute is returned in response to show and list
   operations.

-  **U**. You can update the value of this attribute.

-  **D**. You can delete the value of this attribute.



List networks
~~~~~~~~~~~~~

**GET** /networks

Returns a list of networks with their router:external attributes.

Response codes are same as the normal operation of listing networks.
router:external attribute is visible to all users by default policy
setting.

Regular users are not authorized to create ports on external networks,
however they will be able to see this attribute in their network list.
This is because external networks can be used by any tenant to set an
external gateway for Neutron routers or create floating IPs and
associate them with ports on internal tenant networks.

**Example List networks with router:external attribute: JSON
response**

.. code::

    {
        "networks": [
            {
                "admin_state_up": true,
                "id": "0f38d5ad-10a6-428f-a5fc-825cfe0f1970",
                "name": "net1",
                "router:external": false,
                "shared": false,
                "status": "ACTIVE",
                "subnets": [
                    "25778974-48a8-46e7-8998-9dc8c70d2f06"
                ],
                "tenant_id": "b575417a6c444a6eb5cc3a58eb4f714a"
            },
            {
                "admin_state_up": true,
                "id": "8d05a1b1-297a-46ca-8974-17debf51ca3c",
                "name": "ext_net",
                "router:external": true,
                "shared": false,
                "status": "ACTIVE",
                "subnets": [
                    "2f1fb918-9b0e-4bf9-9a50-6cebbb4db2c5"
                ],
                "tenant_id": "5eb8995cf717462c9df8d1edfa498010"
            }
        ]
    }



Show network details
~~~~~~~~~~~~~~~~~~~~

**GET** /networks/*``network_id``*

Returns details about a specific network, including external networks
attributes.

Response codes are same as the normal operation of listing networks.
router:external attribute is visible to all users including non-admin by
default policy setting.

**Example Show network with external attributes: JSON response**

.. code::

    {
        "network": {
            "admin_state_up": true,
            "id": "8d05a1b1-297a-46ca-8974-17debf51ca3c",
            "name": "ext_net",
            "router:external": true,
            "shared": false,
            "status": "ACTIVE",
            "subnets": [
                "2f1fb918-9b0e-4bf9-9a50-6cebbb4db2c5"
            ],
            "tenant_id": "5eb8995cf717462c9df8d1edfa498010"
        }
    }



Create network
~~~~~~~~~~~~~~

**POST** /networks

Creates a new network using the external network extension attribute.

If the user submitting the request is not allowed to set this attribute,
a 403 Forbidden response will be returned. Usage of this attribute might
be restricted through authorization policies. By the default policy only
admin users can set this attribute.

**Example 4.40. Create network with external attributes: JSON request**

.. code::

    {
        "network": {
            "admin_state_up": true,
            "name": "ext_net",
            "router:external": true
        }
    }



Update network
~~~~~~~~~~~~~~

**PUT** /networks/*``network_id``*

Updates a network, including the external network extension attribute.

If the user submitting the request is not allowed to set this attribute,
a 403 Forbidden response will be returned. Usage of this attribute might
be restricted through authorization policies. By the default policy only
admin users can set this attribute.

**Example Update external attributes for a network: JSON request**

.. code::

    {
       "network":{
          "router:external":true
       }
    }



