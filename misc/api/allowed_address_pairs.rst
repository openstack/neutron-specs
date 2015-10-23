=====================
Allowed address pairs
=====================

The allowed address pair extension extends the port attribute to enable
you to specify arbitrary mac\_address/ip\_address(cidr) pairs that are
allowed to pass through a port regardless of the subnet associated with
the network.

List ports
~~~~~~~~~~

**GET** /ports

Lists ports with their allowed address pair attributes.

Normal Response Code: 200 OK

Error Response Codes: 401 Unauthorized

This operation returns, for each port, its allowed address pair
attributes as well as all the attributes normally returned by the list
port operation.

**Example List ports with allowed address pair attributes: JSON
response**

.. code::

    {
        "ports":[
        {
         "admin_state_up": true,
         "allowed_address_pairs": [{"ip_address": "23.23.23.1",
                                    "mac_address": "fa:16:3e:c4:cd:3f"}],
         "device_id": "",
         "device_owner": "",
         "fixed_ips": [{"ip_address": "10.0.0.2",
                        "subnet_id": "f4145134-b99b-4b18-9940-47239f071923"}],
         "id": "191f5290-3a5a-40ff-b0cb-cd4b115b400e",
         "mac_address": "fa:16:3e:c4:cd:3f",
         "name": "",
         "network_id": "327f2a2f-9d70-417f-ac3a-d3155e25cf25",
         "status": "DOWN",
         "tenant_id": "8462a4d167f84256b7035f4c408c1185"
        },
        {
         "admin_state_up": true,
         "allowed_address_pairs": [],
         "device_id": "",
         "device_owner": "",
         "fixed_ips": [{"ip_address": "10.0.0.3",
                        "subnet_id": "f4145134-b99b-4b18-9940-47239f071923"}],
         "id": "ec2fb9f9-a11b-4791-852d-eb1ab9b27a0e",
         "mac_address": "fa:16:3e:a9:3e:1a",
         "name": "",
         "network_id": "327f2a2f-9d70-417f-ac3a-d3155e25cf25",
         "status": "DOWN",
         "tenant_id": "8462a4d167f84256b7035f4c408c1185"
       }
      ]
    }



**Example List ports with allowed address pair attributes: XML
response**

.. code::

    <?xml version='1.0' encoding='UTF-8'?>
    <ports xmlns="http://openstack.org/quantum/api/v2.0"
        xmlns:quantum="http://openstack.org/quantum/api/v2.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
        <port>
            <status>ACTIVE</status>
            <name/>
            <allowed_address_pairs>
                <allowed_address_pair>
                    <ip_address>23.23.23.1</ip_address>
                    <mac_address>fa:16:3e:c4:cd:3f</mac_address>
                </allowed_address_pair>
            </allowed_address_pairs>
            <admin_state_up quantum:type="bool">True</admin_state_up>
            <network_id>3537e809-8bec-4ae4-a5ab-2c6477760195</network_id>
            <tenant_id>8462a4d167f84256b7035f4c408c1185</tenant_id>
            <device_owner/>
            <mac_address>fa:16:3e:21:4c:2e</mac_address>
            <fixed_ips>
                <fixed_ip>
                    <subnet_id>f4145134-b99b-4b18-9940-47239f071923</subnet_id>
                    <ip_address>10.0.0.21</ip_address>
                </fixed_ip>
            </fixed_ips>
            <id>191f5290-3a5a-40ff-b0cb-cd4b115b400e</id>
            <device_id/>
        </port>
        <port>
            <status>ACTIVE</status>
            <name/>
            <allowed_address_pairs xsi:nil="true"/>
            <admin_state_up quantum:type="bool">True</admin_state_up>
            <network_id>327f2a2f-9d70-417f-ac3a-d3155e25cf25</network_id>
            <tenant_id>8462a4d167f84256b7035f4c408c1185</tenant_id>
            <device_owner/>
            <mac_address>fa:16:3e:a9:3e:1a</mac_address>
            <fixed_ips>
                <fixed_ip>
                    <subnet_id>18cf6972-95cc-4134-a986-843dc7433aa0</subnet_id>
                    <ip_address>10.0.0.5</ip_address>
                </fixed_ip>
            </fixed_ips>
            <id>ec2fb9f9-a11b-4791-852d-eb1ab9b27a0e</id>
            <device_id/>
        </port>
    </ports>



Show port details
~~~~~~~~~~~~~~~~~

**GET** /ports/*``port_id``*

Shows details about a specified port, including allowed address pair
attributes.

Normal Response Code: 200 OK

Error Response Code: 401 Unauthorized, 404 Not Found

**Example Show port with allowed address pair attributes: JSON
response**

.. code::

    {
       "port":
       {
         "admin_state_up": true,
         "allowed_address_pairs": [{"ip_address": "23.23.23.1",
                                    "mac_address": "fa:16:3e:c4:cd:3f"}],
         "device_id": "",
         "device_owner": "",
         "fixed_ips": [{"ip_address": "10.0.0.2",
                        "subnet_id": "f4145134-b99b-4b18-9940-47239f071923"}],
         "id": "191f5290-3a5a-40ff-b0cb-cd4b115b400e",
         "mac_address": "fa:16:3e:c4:cd:3f",
         "name": "",
         "network_id": "327f2a2f-9d70-417f-ac3a-d3155e25cf25",
         "status": "DOWN",
         "tenant_id": "8462a4d167f84256b7035f4c408c1185"
       }
    }



**Example Show port with allowed address pair attributes: XML
response**

.. code::

    <?xml version='1.0' encoding='UTF-8'?>
    <port xmlns="http://openstack.org/quantum/api/v2.0"
           xmlns:quantum="http://openstack.org/quantum/api/v2.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
        <status>ACTIVE</status>
        <name />
        <allowed_address_pairs>
            <allowed_address_pair>
                <ip_address>23.23.23.1</ip_address>
                <mac_address>fa:16:3e:c4:cd:3f</mac_address>
            </allowed_address_pair>
        </allowed_address_pairs>
        <admin_state_up quantum:type="bool">True</admin_state_up>
        <network_id>3537e809-8bec-4ae4-a5ab-2c6477760195</network_id>
        <tenant_id>8462a4d167f84256b7035f4c408c1185</tenant_id>
        <device_owner />
        <mac_address>fa:16:3e:21:4c:2e</mac_address>
        <fixed_ips>
            <fixed_ip>
                <subnet_id>f4145134-b99b-4b18-9940-47239f071923</subnet_id>
                <ip_address>10.0.0.21</ip_address>
            </fixed_ip>
        </fixed_ips>
        <id>191f5290-3a5a-40ff-b0cb-cd4b115b400e</id>
        <device_id />
    </port>



Create port
~~~~~~~~~~~

**POST** /ports

Creates a port and explicitly specifies the allowed address pair
attributes.

Normal Response Code: 201

Error Response Code: 400 Bad Request, 401 Unauthorized, 403 Forbidden

Bad request is returned if an allowed address pair matches the
mac\_address and ip\_address on port.

Note: If the mac\_address field is left out of the body of the request
the mac\_address assigned to the port will be used.

**Example Create port with allowed address pair attributes: JSON
request**

.. code::

    {
     "port":
        {
          "network_id": "3537e809-8bec-4ae4-a5ab-2c6477760195",
          "allowed_address_pairs": [{"ip_address": "10.3.3.3"}]
        }
    }



Update port
~~~~~~~~~~~

**PUT** /ports/*``port_id``*

Updates a port, with new allowed address pair values.

Normal Response Code: 200 OK

Error Response Code: 400 Bad Request, 401 Unauthorized, 404 Not Found,
403 Forbidden

**Example Update allowed address pair attributes for a port: JSON
request**

.. code::

    {
        "port": {
            "allowed_address_pairs":
             [
                {"ip_address": "10.0.0.1"}
             ]
        }
    }



