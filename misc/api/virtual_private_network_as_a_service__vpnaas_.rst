=============================================
Virtual Private Network as a Service (VPNaaS)
=============================================

The VPNaaS extension provides OpenStack tenants with the ability to
extend private networks across the public telecommunication
infrastructure. The capabilities provided by this initial implementation
of the VPNaaS extension are:

-  Site-to-site Virtual Private Network connecting two private networks.

-  Multiple VPN connections per tenant.

-  Supporting IKEv1 policy with 3des, aes-128, aes-256, or aes-192
   encryption.

-  Supporting IPSec policy with 3des, aes-128, aes-256, or aes-192
   encryption, sha1 authentication, ESP, AH, or AH-ESP transform
   protocol, and tunnel or transport mode encapsulation.

-  Dead Peer Detection (DPD) allowing hold, clear, restart, disabled, or
   restart-by-peer actions.

This extension introduces new resources:

-  **service**, a high level object that associates VPN with a specific
   subnet and router.

-  **ikepolicy**, the Internet Key Exchange policy identifying the
   authentication and encryption algorithm used during phase one and
   phase two negotiation of a VPN connection.

-  **ipsecpolicy**, the IP security policy specifying the authentication
   and encryption algorithm, and encapsulation mode used for the
   established VPN connection.

-  **ipsec-site-connection**, has details for the site-to-site IPsec
   connection, including the peer CIDRs, MTU, authentication mode, peer
   address, DPD settings, and status.

Note
~~~~

This extension is **experimental** for the Havana release. The API may
change without backward compatibility.

Concepts
~~~~~~~~

A VPN **service** relates the Virtual Private Network with a specific
subnet and router for a tenant.

An **IKE Policy** is used for phase one and phase two negotiation of the
VPN connection. Configuration selects the authentication and encryption
algorithm used to establish a connection.

An **IPsec Policy** is used to specify the encryption algorithm,
transform protocol, and mode (tunnel/transport) for the VPN connection.

A VPN **connection** represents the IPsec tunnel established between two
sites for the tenant. This contains configuration settings specifying
the policies used, peer information, MTU, and the DPD actions to take.

High-level flow
~~~~~~~~~~~~~~~

The high-level task flow for using VPNaaS API to configure a
site-to-site Virtual Private Network is as follows:

#. The tenant creates a VPN service specifying the router and subnet.

#. The tenant creates an IKE Policy.

#. The tenant creates an IPsec Policy.

#. The tenant creates a VPN connection, specifying the VPN service, peer
   information, and IKE and IPsec policies.

VPN services
~~~~~~~~~~~~

Manage a tenant's VPN service through this extension.

**Table VPN Service Attributes**

Attribute

Type

Required

CRUD `:sup:`[a]` <#ftn.vpnaas_service_crud_note>`__

Default value

Validation constraints

Notes

id

uuid-str

N/A

R

generated

N/A

Unique identifier for the VPN Service object.

tenant\_id

uuid-str

Yes

CR

Derived from Authentication token

valid tenant\_id

Owner of the VPN service. Only admin users can specify a tenant
identifier other than their own.

name

String

No

CRU

None

N/A

Human readable name for the VPN service. Does not have to be unique.

description

String

No

CRU

None

N/A

Human readable description for the VPN service.

status

String

N/A

R

N/A

N/A

Indicates whether IPsec VPN service is currently operational. Possible
values include: ACTIVE, DOWN, BUILD, ERROR, PENDING\_CREATE,
PENDING\_UPDATE, or PENDING\_DELETE.

admin\_state\_up

Bool

N/A

CRU

true

{true \false }

Administrative state of the vpnservice. If false (down), port does not
forward packets.

subnet\_id

uuid-str

Yes

CR

N/A

valid subnet ID

The subnet on which the tenant wants the VPN service. This may be
extended in the future to support multiple subnets.

router\_id

uuid-str

Yes

CR

N/A

valid router ID

Router ID to which the VPN service is inserted. This may change in the
future, when router level insertion is available.

-  **`:sup:`[a]` <#vpnaas_service_crud_note>`__\ C**. Use the attribute
   in create operations.

-  **R**. This attribute is returned in response to show and list
   operations.

-  **U**. You can update the value of this attribute.

-  **D**. You can delete the value of this attribute.



List VPN services
^^^^^^^^^^^^^^^^^

**GET** /vpn/vpnservices

Lists VPN services.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example List VPN Services: Request**

.. code::

    GET /v2.0/vpn/vpnservices.json
    User-Agent: python-neutronclient
    Accept: application/json


**Example List VPN Services: Response**

.. code::

    {
      "vpnservices": [
        {
          "router_id": "ec8619be-0ba8-4955-8835-3b49ddb76f89",
          "status": "PENDING_CREATE",
          "name": "myservice",
          "admin_state_up": true,
          "subnet_id": "f4fb4528-ed93-467c-a57b-11c7ea9f963e",
          "tenant_id": "ccb81365fe36411a9011e90491fe1330",
          "id": "9faaf49f-dd89-4e39-a8c6-101839aa49bc",
          "description": ""
        }
      ]
    }



Show VPN service details
^^^^^^^^^^^^^^^^^^^^^^^^

**GET** /vpn/vpnservices/*``service-id``*

Shows details about a specified VPN service.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403), Not Found
(404)

This operation does not require a request body.

This operation returns a response body.

**Example Show VPN Service: Request**

.. code::

    GET /v2.0/vpn/vpnservices/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json

**Example Show VPN Service: Response**

.. code::

    {
      "vpnservice": {
        "router_id": "ec8619be-0ba8-4955-8835-3b49ddb76f89",
        "status": "PENDING_CREATE",
        "name": "myservice",
        "admin_state_up": true,
        "subnet_id": "f4fb4528-ed93-467c-a57b-11c7ea9f963e",
        "tenant_id": "ccb81365fe36411a9011e90491fe1330",
        "id": "9faaf49f-dd89-4e39-a8c6-101839aa49bc",
        "description": ""
      }
    }



Create VPN service
^^^^^^^^^^^^^^^^^^

**POST** /vpn/vpnservices

Creates a VPN service.

Normal Response Code: 201

Error Response Codes: Unauthorized (401), Bad Request (400)

This operation requires a request body.

This operation returns a response body.

**Example Create VPN Service: Request**

.. code::

    POST /v2.0/vpn/vpnservices.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
      "vpnservice": {
        "subnet_id": "f4fb4528-ed93-467c-a57b-11c7ea9f963e",
        "router_id": "ec8619be-0ba8-4955-8835-3b49ddb76f89",
        "name": "myservice",
        "admin_state_up": true
      }
    }



**Example Create VPN: Response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "vpnservice": {
        "router_id": "ec8619be-0ba8-4955-8835-3b49ddb76f89",
        "status": "PENDING_CREATE",
        "name": "myservice",
        "admin_state_up": true,
        "subnet_id": "f4fb4528-ed93-467c-a57b-11c7ea9f963e",
        "tenant_id": "ccb81365fe36411a9011e90491fe1330",
        "id": "9faaf49f-dd89-4e39-a8c6-101839aa49bc",
        "description": ""
      }
    }



Update VPN service
^^^^^^^^^^^^^^^^^^

**PUT** /vpn/vpnservices/*``service-id``*

Updates a VPN service, provided status is not indicating a PENDING\_\*
state.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Bad Request (400), Not Found
(404)

**Example Update VPN Service: Request**

.. code::

    PUT /v2.0/vpn/vpnservices/41bfef97-af4e-4f6b-a5d3-4678859d2485.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
      "vpnservice": {
        "description": "Updated description"
      }
    }



**Example Update VPN Service: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "vpnservice": {
        "router_id": "881b7b30-4efb-407e-a162-5630a7af3595",
        "status": "ACTIVE",
        "name": "myvpn",
        "admin_state_up": true,
        "subnet_id": "25f8a35c-82d5-4f55-a45b-6965936b33f6",
        "tenant_id": "26de9cd6cae94c8cb9f79d660d628e1f",
        "id": "41bfef97-af4e-4f6b-a5d3-4678859d2485",
        "description": "Updated description"
      }
    }



Delete VPN service
^^^^^^^^^^^^^^^^^^

**DELETE** /vpn/vpnservices/*``service-id``*

Deletes a VPN service.

Normal Response Code: 204

Error Response Codes: Unauthorized (401), Not Found (404), Conflict
(409)

This operation does not require a request body.

This operation does not return a response body.

**Example Delete VPN Service: Request**

.. code::

    DELETE /v2.0/vpn/vpnservices/1be5e5f7-c45e-49ba-85da-156575b60d50.json
    User-Agent: python-neutronclient
    Accept: application/json

**Example Delete VPN Service: Response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0


IKE policies
~~~~~~~~~~~~

Manage IKE policies through the VPN as a Service extension.

**Table IKE Policy Attributes**

Attribute

Type

Required

CRUD `:sup:`[a]` <#ftn.vpnaas_ikepolicy_crud_note>`__

Default value

Validation constraints

Notes

id

uuid-str

N/A

R

generated

N/A

Unique identifier for the IKE policy.

tenant\_id

uuid-str

Yes

CR

None

valid tenant\_id

Unique identifier for owner of the VPN service.

name

string

yes

CRU

None

N/A

Friendly name for the IKE policy.

description

string

no

CRU

None

N/A

Description of the IKE policy.

auth\_algorithm

string

no

CRU

sha1

N/A

Authentication Hash algorithms: sha1.

encryption\_algorithm

string

no

CRU

aes-128

N/A

Encryption Algorithms: 3des, aes-128, aes-256, aes-192, etc.

phase1\_negotiation\_mode

string

no

CRU

Main Mode

N/A

IKE mode: Main Mode.

pfs

string

no

CRU

Group5

N/A

Perfect Forward Secrecy: Group2, Group5, or Group14.

ike\_version

string

no

CRU

v1

N/A

Version: v1 or v2.

lifetime

dict

no

CRU

units: seconds, value: 3600.

Dictionary should be in this form: {'units': 'seconds', 'value': 2000}.
Value is a positive integer.

Lifetime of the SA. Units in 'seconds'. Either units or value may be
omitted.

-  **`:sup:`[a]` <#vpnaas_ikepolicy_crud_note>`__\ C**. Use the
   attribute in create operations.

-  **R**. This attribute is returned in response to show and list
   operations.

-  **U**. You can update the value of this attribute.

-  **D**. You can delete the value of this attribute.


List IKE policies
^^^^^^^^^^^^^^^^^

**GET** /vpn/ikepolicies

Lists IKE policies.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example List IKE Policies: Request**

.. code::

    GET /v2.0/vpn/ikepolicies.json
    User-Agent: python-neutronclient
    Accept: application/json


**Example List IKE Policies: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ikepolicies": [
        {
          "name": "ikepolicy1",
          "tenant_id": "ccb81365fe36411a9011e90491fe1330",
          "auth_algorithm": "sha1",
          "encryption_algorithm": "aes-256",
          "pfs": "group5",
          "phase1_negotiation_mode": "main",
          "lifetime": {
            "units": "seconds",
            "value": 3600
          },
          "ike_version": "v1",
          "id": "5522aff7-1b3c-48dd-9c3c-b50f016b73db",
          "description": ""
        }
      ]
    }



Show IKE policy details
^^^^^^^^^^^^^^^^^^^^^^^

**GET** /vpn/ikepolicies/*``ikepolicy-id``*

Shows details for a specified IKE policy.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403), Not Found
(404)

This operation does not require a request body.

This operation returns a response body.

**Example Show IKE Policy: Request**

.. code::

    GET /v2.0/vpn/ikepolicies/5522aff7-1b3c-48dd-9c3c-b50f016b73db.json
    User-Agent: python-neutronclient
    Accept: application/json


**Example Show IKE Policy: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ikepolicy": {
        "name": "ikepolicy1",
        "tenant_id": "ccb81365fe36411a9011e90491fe1330",
        "auth_algorithm": "sha1",
        "encryption_algorithm": "aes-256",
        "pfs": "group5",
        "phase1_negotiation_mode": "main",
        "lifetime": {
          "units": "seconds",
          "value": 3600
        },
        "ike_version": "v1",
        "id": "5522aff7-1b3c-48dd-9c3c-b50f016b73db",
        "description": ""
      }
    }



Create IKE policy
^^^^^^^^^^^^^^^^^

**POST** /vpn/ikepolicies

Creates an IKE policy.

Normal Response Code: 201

Error Response Codes: Unauthorized (401), Bad Request (400)

This operation requires a request body.

This operation returns a response body.

**Example Create IKE Policy: Request**

.. code::

    POST /v2.0/vpn/ikepolicies.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
      "ikepolicy": {
        "phase1_negotiation_mode": "main",
        "auth_algorithm": "sha1",
        "encryption_algorithm": "aes-128",
        "pfs": "group5",
        "lifetime": {
          "units": "seconds",
          "value": 7200
        },
        "ike_version": "v1",
        "name": "ikepolicy1"
      }
    }



**Example Create IKE Policy: Response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ikepolicy": {
        "name": "ikepolicy1",
        "tenant_id": "ccb81365fe36411a9011e90491fe1330",
        "auth_algorithm": "sha1",
        "encryption_algorithm": "aes-128",
        "pfs": "group5",
        "phase1_negotiation_mode": "main",
        "lifetime": {
          "units": "seconds",
          "value": 7200
        },
        "ike_version": "v1",
        "id": "5522aff7-1b3c-48dd-9c3c-b50f016b73db",
        "description": ""
      }
    }



Update IKE policy
^^^^^^^^^^^^^^^^^

**PUT** /vpn/ikepolicies/*``ikepolicy-id``*

Updates an IKE policy.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Bad Request (400), Not Found
(404)

**Example Update IKE Policy: Request**

.. code::

    PUT /v2.0/vpn/ikepolicies/5522aff7-1b3c-48dd-9c3c-b50f016b73db.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
      "ikepolicy": {
        "encryption_algorithm": "aes-256"
      }
    }



**Example Update IKE Policy: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ikepolicy": {
        "name": "ikepolicy1",
        "tenant_id": "ccb81365fe36411a9011e90491fe1330",
        "auth_algorithm": "sha1",
        "encryption_algorithm": "aes-256",
        "pfs": "group5",
        "phase1_negotiation_mode": "main",
        "lifetime": {
          "units": "seconds",
          "value": 3600
        },
        "ike_version": "v1",
        "id": "5522aff7-1b3c-48dd-9c3c-b50f016b73db",
        "description": ""
      }
    }



Delete IKE policy
^^^^^^^^^^^^^^^^^

**DELETE** /vpn/ikepolicies/*``ikepolicy-id``*

Deletes an IKE policy.

Normal Response Code: 204

Error Response Codes: Unauthorized (401), Not Found (404), Conflict
(409)

This operation does not require a request body.

This operation does not return a response body.

**Example Delete IKE Policy: Request**

.. code::

    DELETE /v2.0/vpn/ikepolicies/5522aff7-1b3c-48dd-9c3c-b50f016b73db.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Delete IKE Policy: Response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0



IPSec policies
~~~~~~~~~~~~~~

Manage IPSec policies through the VPN as a Service extension.

**Table IPSec Policy Attributes**

Attribute

Type

Required

CRUD `:sup:`[a]` <#ftn.vpnaas_ipsec_crud_note>`__

Default value

Validation constraints

Notes

id

uuid-str

N/A

R

generated

N/A

Unique identifier for the IPsec policy.

tenant\_id

uuid-str

Yes

CR

None

valid tenant\_id

Unique identifier for owner of the VPN service.

name

string

yes

CRU

None

N/A

Friendly name for the IPsec policy.

description

string

no

CRU

None

N/A

Description of the IPSec policy.

transform\_protocol

string

no

CRU

ESP

N/A

Transform protocol used: ESP, AH, or AH-ESP.

encapsulation\_mode

string

no

CRU

tunnel

N/A

Encapsulation mode: tunnel or transport.

auth\_algorithm

string

no

CRU

sha1

N/A

Authentication algorithm: sha1.

encryption\_algorithm

string

no

CRU

aes-128

N/A

Encryption Algorithms: 3des, aes-128, aes-256, or aes-192.

pfs

string

no

CRU

group5

N/A

Perfect Forward Secrecy: group2, group5, or group14.

lifetime

dict

no

CRU

units: seconds, value: 3600.

Dictionary should be in this form: {'units': 'seconds', 'value': 2000}.
Value is a positive integer.

Lifetime of the SA. Units in 'seconds'. Either units or value may be
omitted.

-  **`:sup:`[a]` <#vpnaas_ipsec_crud_note>`__\ C**. Use the attribute in
   create operations.

-  **R**. This attribute is returned in response to show and list
   operations.

-  **U**. You can update the value of this attribute.

-  **D**. You can delete the value of this attribute.



List IPSec policies
^^^^^^^^^^^^^^^^^^^

**GET** /vpn/ipsecpolicies

Lists IPSec policies.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example List IPSec Policies: Request**

.. code::

    GET /v2.0/vpn/ipsecpolicies.json
    User-Agent: python-neutronclient
    Accept: application/json


**Example List IPSec Policies: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ipsecpolicies": [
        {
          "name": "ipsecpolicy1",
          "transform_protocol": "esp",
          "auth_algorithm": "sha1",
          "encapsulation_mode": "tunnel",
          "encryption_algorithm": "aes-128",
          "pfs": "group14",
          "tenant_id": "ccb81365fe36411a9011e90491fe1330",
          "lifetime": {
            "units": "seconds",
            "value": 3600
          },
          "id": "5291b189-fd84-46e5-84bd-78f40c05d69c",
          "description": ""
        }
      ]
    }



Show IPSec policy details
^^^^^^^^^^^^^^^^^^^^^^^^^

**GET** /vpn/ipsecpolicies/*``ipsecpolicy-id``*

Shows details for a specified IPSec policy.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403), Not Found
(404)

This operation does not require a request body.

This operation returns a response body.

**Example Show IPSec policy: Request**

.. code::

    GET /v2.0/vpn/ipsecpolicies/5291b189-fd84-46e5-84bd-78f40c05d69c.json
    User-Agent: python-neutronclient
    Accept: application/json


**Example Show IPSec policy: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ipsecpolicy": {
        "name": "ipsecpolicy1",
        "transform_protocol": "esp",
        "auth_algorithm": "sha1",
        "encapsulation_mode": "tunnel",
        "encryption_algorithm": "aes-128",
        "pfs": "group14",
        "tenant_id": "ccb81365fe36411a9011e90491fe1330",
        "lifetime": {
          "units": "seconds",
          "value": 3600
        },
        "id": "5291b189-fd84-46e5-84bd-78f40c05d69c",
        "description": ""
      }
    }



Create IPSec Policy
^^^^^^^^^^^^^^^^^^^

**POST** /vpn/ipsecpolicies

Creates an IPSec policy.

Normal Response Code: 201

Error Response Codes: Unauthorized (401), Bad Request (400)

This operation requires a request body.

This operation returns a response body.

**Example Create IPSec policy: Request**

.. code::

    POST /v2.0/vpn/ipsecpolicies.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
      "ipsecpolicy": {
        "name": "ipsecpolicy1",
        "transform_protocol": "esp",
        "auth_algorithm": "sha1",
        "encapsulation_mode": "tunnel",
        "encryption_algorithm": "aes-128",
        "pfs": "group5",
        "lifetime": {
          "units": "seconds",
          "value": 7200
        }
      }
    }



**Example Create IPSec policy: Response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ipsecpolicy": {
        "name": "ipsecpolicy1",
        "transform_protocol": "esp",
        "auth_algorithm": "sha1",
        "encapsulation_mode": "tunnel",
        "encryption_algorithm": "aes-128",
        "pfs": "group5",
        "tenant_id": "ccb81365fe36411a9011e90491fe1330",
        "lifetime": {
          "units": "seconds",
          "value": 7200
        },
        "id": "5291b189-fd84-46e5-84bd-78f40c05d69c",
        "description": ""
      }
    }



Update IPSec Policy
^^^^^^^^^^^^^^^^^^^

**PUT** /vpn/ipsecpolicies/*``ipsecpolicy-id``*

Updates an IPSec policy.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Bad Request (400), Not Found
(404)

**Example Update IPSec policy: Request**

.. code::

    PUT /v2.0/vpn/ipsecpolicies/5291b189-fd84-46e5-84bd-78f40c05d69c.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
      "ipsecpolicy": {
        "pfs": "group14"
      }
    }



**Example Update IPSec policy: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ipsecpolicy": {
        "name": "ipsecpolicy1",
        "transform_protocol": "esp",
        "auth_algorithm": "sha1",
        "encapsulation_mode": "tunnel",
        "encryption_algorithm": "aes-128",
        "pfs": "group14",
        "tenant_id": "ccb81365fe36411a9011e90491fe1330",
        "lifetime": {
          "units": "seconds",
          "value": 3600
        },
        "id": "5291b189-fd84-46e5-84bd-78f40c05d69c",
        "description": ""
      }
    }



Delete IPSec policy
^^^^^^^^^^^^^^^^^^^

**DELETE** /vpn/ipsecpolicies/*``ipsecpolicy-id``*

Deletes an IPSec policy.

Normal Response Code: 204

Error Response Codes: Unauthorized (401), Not Found (404), Conflict
(409)

This operation does not require a request body.

This operation does not return a response body.

**Example Delete IPSec policy: Request**

.. code::

    DELETE /v2.0/vpn/ipsecpolicies/5291b189-fd84-46e5-84bd-78f40c05d69c.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Delete IPSec policy: Response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0



IPSec site connections
~~~~~~~~~~~~~~~~~~~~~~

Manage IPSec site-to-site connections through the VPN as a Service
extension.

**Table IPSec site connection attributes**

Attribute

Type

Required

CRUD `:sup:`[a]` <#ftn.vpnaas_ipsec_site_connection_crud_note>`__

Default Value

Validation Constraints

Notes

id

uuid-str

N/A

R

generated

N/A

Unique identifier for the IPSec site-to-site connection.

tenant\_id

uuid-str

Yes

CR

None

valid tenant\_id

Unique identifier for owner of the VPN service.

name

string

no

CRU

None

N/A

Name for IPSec site-to-site connection.

description

string

no

CRU

None

N/A

Description of the IPSec site-to-site connection.

peer\_address

string

yes

CRU

N/A

N/A

Peer gateway public IPv4/IPv6 address or FQDN.

peer\_id

string

yes

CRU

N/A

N/A

Peer router identity for authentication. Can be IPv4/IPv6 address,
e-mail address, key id, or FQDN.

peer\_cidrs

list[string]

yes

CRU

N/A

unique list of valid cidr in the form <net\_address>/<prefix>

Peer private CIDRs.

route\_mode

string

no

R

static

static

Route mode: static. This will be extended in the future.

mtu

integer

no

CRU

1500

Integer. Minimum is 68 for IPv4 and 1280 for IPv6.

Maximum Transmission Unit to address fragmentation.

auth\_mode

string

no

R

psk

psk/certs

Authentication mode: PSK or certificate.

psk

string

yes

CRU

N/A

NO

Pre Shared Key: any string.

initiator

string

no

CRU

bi-directional

bi-directional / response-only

Whether this VPN can only respond to connections or can initiate as
well.

admin\_state\_up

bool

N/A

CRU

TRUE

true / false

Administrative state of VPN connection. If false (down), VPN connection
does not forward packets.

status

string

N/A

R

N/A

N/A

Indicates whether VPN connection is currently operational. Possible
values include: ACTIVE, DOWN, BUILD, ERROR, PENDING\_CREATE,
PENDING\_UPDATE, or PENDING\_DELETE.

ikepolicy\_id

uuid

yes

CR

N/A

Unique identifier of IKE policy

Unique identifier of IKE policy.

ipsecpolicy\_id

uuid

yes

CR

N/A

Unique identifier of IPSec policy

Unique identifier of IPSec policy.

vpnservice\_id

uuid

yes

CR

N/A

Unique identifier of VPN service

Unique identifier of VPN service.

dpd

dict

no

CRU

action: hold, interval: 30, timeout: 120

Dictionary should be in this form: {'action': 'clear', 'interval': 20,
'timeout': 60}. Interval is positive integer. Timeout is greater than
interval.

Dead Peer Detection protocol controls. Action: clear, hold, restart,
disabled, or restart-by-peer. Interval and timeout in seconds.

-  **`:sup:`[a]` <#vpnaas_ipsec_site_connection_crud_note>`__\ C**. Use
   the attribute in create operations.

-  **R**. This attribute is returned in response to show and list
   operations.

-  **U**. You can update the value of this attribute.

-  **D**. You can delete the value of this attribute.



List IPSec site connections
^^^^^^^^^^^^^^^^^^^^^^^^^^^

**GET**

/vpn/ipsec-site-connections

Lists the IPSec site-to-site connections.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example List IPSec site connections: Request**

.. code::

    GET /v2.0/vpn/ipsec-site-connections.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example List IPSec site connections: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ipsec_site_connections": [
        {
          "status": "PENDING_CREATE",
          "psk": "secret",
          "initiator": "bi-directional",
          "name": "vpnconnection1",
          "admin_state_up": true,
          "tenant_id": "ccb81365fe36411a9011e90491fe1330",
          "description": "",
          "auth_mode": "psk",
          "peer_cidrs": [
            "10.1.0.0/24"
          ],
          "mtu": 1500,
          "ikepolicy_id": "bf5612ac-15fb-460c-9b3d-6453da2fafa2",
          "dpd": {
            "action": "hold",
            "interval": 30,
            "timeout": 120
          },
          "route_mode": "static",
          "vpnservice_id": "c2f3178d-5530-4c4a-89fc-050ecd552636",
          "peer_address": "172.24.4.226",
          "peer_id": "172.24.4.226",
          "id": "cbc152a0-7e93-4f98-9f04-b085a4bf2511",
          "ipsecpolicy_id": "8ba867b2-67eb-4835-bb61-c226804a1584"
        }
      ]
    }



Show IPSec site connection details
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**GET**

/vpn/ipsec-site-connections/*``connection-id``*

Shows details about a specified IPSec site-to-site connection.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403), Not Found
(404)

This operation does not require a request body.

This operation returns a response body.

**Example Show IPSec site connection: Request**

.. code::

    GET /v2.0/vpn/ipsec-site-connections/cbc152a0-7e93-4f98-9f04-b085a4bf2511.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Show IPSec site connection: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ipsec_site_connection": {
        "status": "PENDING_CREATE",
        "psk": "secret",
        "initiator": "bi-directional",
        "name": "vpnconnection1",
        "admin_state_up": true,
        "tenant_id": "ccb81365fe36411a9011e90491fe1330",
        "description": "",
        "auth_mode": "psk",
        "peer_cidrs": [
          "10.1.0.0/24"
        ],
        "mtu": 1500,
        "ikepolicy_id": "bf5612ac-15fb-460c-9b3d-6453da2fafa2",
        "dpd": {
          "action": "hold",
          "interval": 30,
          "timeout": 120
        },
        "route_mode": "static",
        "vpnservice_id": "c2f3178d-5530-4c4a-89fc-050ecd552636",
        "peer_address": "172.24.4.226",
        "peer_id": "172.24.4.226",
        "id": "cbc152a0-7e93-4f98-9f04-b085a4bf2511",
        "ipsecpolicy_id": "8ba867b2-67eb-4835-bb61-c226804a1584"
      }
    }



Create IPSec site connection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**POST**

/vpn/ipsec-site-connections

Creates an IPSec site connection.

Normal Response Code: 201

Error Response Codes: Unauthorized (401), Bad Request (400)

This operation requires a request body.

This operation returns a response body.

**Example Create IPSec site connection: Request**

.. code::

    POST /v2.0/vpn/ipsec-site-connections.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
      "ipsec_site_connection": {
        "psk": "secret",
        "initiator": "bi-directional",
        "ipsecpolicy_id": "22b8abdc-e822-45b3-90dd-f2c8512acfa5",
        "admin_state_up": true,
        "peer_cidrs": [
          "10.2.0.0/24"
        ],
        "mtu": "1500",
        "ikepolicy_id": "d3f373dc-0708-4224-b6f8-676adf27dab8",
        "dpd": {
          "action": "disabled",
          "interval": 60,
          "timeout": 240
        },
        "vpnservice_id": "7b347d20-6fa3-4e22-b744-c49ee235ae4f",
        "peer_address": "172.24.4.233",
        "peer_id": "172.24.4.233",
        "name": "vpnconnection1"
      }
    }



**Example Create IPSec site connection: Response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ipsec_site_connection": {
        "status": "PENDING_CREATE",
        "psk": "secret",
        "initiator": "bi-directional",
        "name": "vpnconnection1",
        "admin_state_up": true,
        "tenant_id": "b6887d0b45b54a249b2ce3dee01caa47",
        "description": "",
        "auth_mode": "psk",
        "peer_cidrs": [
          "10.2.0.0/24"
        ],
        "mtu": 1500,
        "ikepolicy_id": "d3f373dc-0708-4224-b6f8-676adf27dab8",
        "dpd": {
          "action": "disabled",
          "interval": 60,
          "timeout": 240
        },
        "route_mode": "static",
        "vpnservice_id": "7b347d20-6fa3-4e22-b744-c49ee235ae4f",
        "peer_address": "172.24.4.233",
        "peer_id": "172.24.4.233",
        "id": "af44dfd7-cf91-4451-be57-cd4fdd96b5dc",
        "ipsecpolicy_id": "22b8abdc-e822-45b3-90dd-f2c8512acfa5"
      }
    }



Update IPSec site connection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**PUT**

/vpn/ipsec-site-connections/*``connection-id``*

Updates an IPSec site-to-site connection, provided status is not
indicating a PENDING\_\* state.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Bad Request (400), Not Found
(404)

**Example Update IPSec site connection: Request**

.. code::

    PUT /v2.0/vpn/ipsec-site-connections/f7cf7305-f491-45f4-ad9c-8e7240fe3d72.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
      "ipsec_site_connection": {
        "mtu": "2000"
      }
    }



**Example Update IPSec site connection: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
      "ipsec_site_connection": {
        "status": "DOWN",
        "psk": "secret",
        "initiator": "bi-directional",
        "name": "vpnconnection1",
        "admin_state_up": true,
        "tenant_id": "26de9cd6cae94c8cb9f79d660d628e1f",
        "description": "",
        "auth_mode": "psk",
        "peer_cidrs": [
          "10.2.0.0/24"
        ],
        "mtu": 2000,
        "ikepolicy_id": "771f081c-5ec8-4f9a-b041-015dfb7fbbe2",
        "dpd": {
          "action": "hold",
          "interval": 30,
          "timeout": 120
        },
        "route_mode": "static",
        "vpnservice_id": "41bfef97-af4e-4f6b-a5d3-4678859d2485",
        "peer_address": "172.24.4.233",
        "peer_id": "172.24.4.233",
        "id": "f7cf7305-f491-45f4-ad9c-8e7240fe3d72",
        "ipsecpolicy_id": "9958d4fe-3719-4e8c-84e7-9893895b76b4"
      }
    }



Delete IPSec site connection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**DELETE**

/vpn/ipsec-site-connections/*``connection-id``*

Deletes an IPSec site-to-site connection.

Normal Response Code: 204

Error Response Codes: Unauthorized (401), Not Found (404), Conflict
(409)

This operation does not require a request body.

This operation does not return a response body.

**Example Delete IPSec site connection: Request**

.. code::

    DELETE /v2.0/vpn/ipsec-site-connections/cbc152a0-7e93-4f98-9f04-b085a4bf2511.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Delete IPSec site connection: Response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0



