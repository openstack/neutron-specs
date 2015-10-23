===================================
Extra DHCP options (extra-dhcp-opt)
===================================

The DHCP options extension allows adding DHCP options that are
associated to a Neutron port. They are tagged such that they can be
associated from the hosts file to designate a specific network interface
and port. The DHCP tag scheme used to associate options to the host
files is the port\_id (UUID - in the form of ``8-4-4-4-12`` for a total
of 36 characters), these associate options to a Neutron port and its
network. The Dynamic Host Configuration Protocol (DHCP) provides a
framework for passing configuration information to hosts on a TCP/IP
network. Configuration parameters and other information are carried in
tagged data items that are stored in the 'options' field of the DHCP
message.

You can specify a DHCP options when defining or updating a port by
specifying the extra\_dhcp\_opts tag and providing its options as name
value pairs, such as, opt\_name='bootfile-name',
opt\_value='pxelinux.0'.

Concepts
~~~~~~~~

The ``extra-dhcp-opt`` extension is an attribute extension which adds
the following set of attributes to the **port** resource:

-  extra-dhcp-opt:opt\_name - Specified the DHCP option that this is
   defined as mapped to this port resource. Examples are
   ``bootfile-name``, ``server-ip-address``, ``tftp-server``, etc..

-  extra-dhcp-opt:opt\_value - Identifies the value associated with the
   opt\_name. These are handled in opt\_name, opt\_value pairs only.
   value\_opt can be any text string depending upon the name.

The actual semantics of ``extra-dhcp-opt`` attributes depend on the name
of the dhcp option being used that defines the vendor extension to DHCP.
For example reference RFC: http://tools.ietf.org/html/rfc2132, contains
specific detail on BOOTP Vendor Extensions.

List ports
~~~~~~~~~~

**GET** /ports

Lists ports with attributes.

Normal response Code: 200 OK

Error response Codes: 401 Unauthorized

This operation returns all the ports defined in Neutron that to which
this user has access.

**Example List ports with extra\_dhcp\_opts: JSON response**

.. code::

    {
      "ports": [
      {
        "status": "DOWN",
        "binding:host_id": null,
        "name": "",
        "allowed_address_pairs": [],
        "admin_state_up": true,
        "network_id": "87733bcc-8144-41b1-bb6b-d011d7a5168e",
        "tenant_id": "7ea98790cd854fb5a82ef3d41e5c156b",
        "extra_dhcp_opts": [{"opt_value": "testfile.1", "opt_name": "bootfile-name"}, {"opt_value": "123.123.123.45", "opt_name": "server-ip-address"}, {"opt_value": "123.123.123.123", "opt_name": "tftp-server"}],
        "binding:vif_type": "ovs",
        "device_owner": "",
        "binding:capabilities": {"port_filter": true},
        "mac_address": "fa:16:3e:52:92:3a",
        "fixed_ips": [{"subnet_id": "99a8aea3-b9da-409d-a5e5-f45338ceb4d3", "ip_address": "172.24.4.228"}],
        "id": "3c0c7a37-690a-43a8-8088-5d4c2c7f8484",
        "security_groups": ["9bf6f19a-ba4a-470f-b8ce-28c9ad66556c"],
        "device_id": ""
      },
      {
        "status": "ACTIVE",
        "binding:host_id": null,
        "name": "",
        "allowed_address_pairs": [],
        "admin_state_up": true,
        "network_id": "87733bcc-8144-41b1-bb6b-d011d7a5168e",
        "tenant_id": "7ea98790cd854fb5a82ef3d41e5c156b",
        "extra_dhcp_opts": [],
        "binding:vif_type": "ovs",
        "device_owner": "compute:probe",
        "binding:capabilities": {"port_filter": true},
        "mac_address": "fa:16:3e:49:56:07",
        "fixed_ips": [{"subnet_id": "99a8aea3-b9da-409d-a5e5-f45338ceb4d3", "ip_address": "172.24.4.227"}],
        "id": "5212d40a-c2f5-4a5d-ad18-694658047654",
        "security_groups": ["9bf6f19a-ba4a-470f-b8ce-28c9ad66556c"],
        "device_id": "zvm2"
      }
      ]
    }



Show port details
~~~~~~~~~~~~~~~~~

**GET** /ports/*``port_id``*

Shows details about a specified port, including ``extra-dhcp-opt``
attributes.

Normal response Code: 200 OK

Error response Code: 401 Unauthorized, 404 Not Found

This operation returns, for the port specified in the request URI, its
port attributes, including the extra\_dhcp\_opts attributes.

**Example Show port details with extra-dhcp-opt attributes: JSON
response**

.. code::

    {
        "port":
        {
            "status": "DOWN",
            "binding:host_id": null,
            "name": "",
            "allowed_address_pairs": [],
            "admin_state_up": true,
            "network_id": "87733bcc-8144-41b1-bb6b-d011d7a5168e",
            "tenant_id": "7ea98790cd854fb5a82ef3d41e5c156b",
            "extra_dhcp_opts": [
                {"opt_value": "testfile.1","opt_name": "bootfile-name"},
                {"opt_value": "123.123.123.123", "opt_name": "tftp-server"},
                {"opt_value": "123.123.123.45", "opt_name": "server-ip-address"}
            ],
            "binding:vif_type": "ovs",
            "device_owner": "",
            "binding:capabilities": {"port_filter": true},
            "mac_address": "fa:16:3e:52:92:3a",
            "fixed_ips": [{"subnet_id": "99a8aea3-b9da-409d-a5e5-f45338ceb4d3", "ip_address": "172.24.4.228"}],
            "id": "3c0c7a37-690a-43a8-8088-5d4c2c7f8484",
            "security_groups": ["9bf6f19a-ba4a-470f-b8ce-28c9ad66556c"],
            "device_id": ""
         }
    }



Create port
~~~~~~~~~~~

**POST** /ports

Creates a port and explicitly specifies attributes with the
``extra-dhcp-opt`` extension attributes.

Normal response Code: 200 OK

Error response Code: 401 Unauthorized.

This operation returns, for the port specified in the request URI, its
port attributes, including the extra\_dhcp\_opts attributes.

**Example Create port with extra-dhcp-opt attributes: JSON
request**

.. code::

    {
        "port":
        {
            "network_id": "87733bcc-8144-41b1-bb6b-d011d7a5168e",
            "extra_dhcp_opts": [
                {"opt_value": "pxelinux.0", "opt_name": "bootfile-name"},
                {"opt_value": "123.123.123.123", "opt_name": "tftp-server"},
                {"opt_value": "123.123.123.45", "opt_name": "server-ip-address"}
            ],
            "fixed_ips": [{"subnet_id": "99a8aea3-b9da-409d-a5e5-f45338ceb4d3", "ip_address": "172.24.4.230"}],
            "admin_state_up": true
        }
    }



**Example Create port with extra-dhcp-opt attributes: JSON
response**

.. code::

    {
        "port":
        {
            "status": "DOWN",
            "binding:host_id": null,
            "name": "",
            "allowed_address_pairs": [],
            "admin_state_up": true,
            "network_id": "87733bcc-8144-41b1-bb6b-d011d7a5168e",
            "tenant_id": "7ea98790cd854fb5a82ef3d41e5c156b",
            "extra_dhcp_opts": [
                {"opt_value": "123.123.123.123", "opt_name": "tftp-server"},
                {"opt_value": "pxelinux.0", "opt_name": "bootfile-name"},
                {"opt_value": "123.123.123.45", "opt_name": "server-ip-address"}
            ],
            "binding:vif_type": "ovs",
            "device_owner": "",
            "binding:capabilities": {"port_filter": true},
            "mac_address": "fa:16:3e:43:3c:b7",
            "fixed_ips": [{"subnet_id": "99a8aea3-b9da-409d-a5e5-f45338ceb4d3", "ip_address": "172.24.4.230"}],
            "id": "055d27c0-0194-4782-be45-275ff2c95c61",
            "security_groups": ["9bf6f19a-ba4a-470f-b8ce-28c9ad66556c"],
            "device_id": ""
        }
    }



Update port
~~~~~~~~~~~

**PUT** /ports/*``port_id``*

Updates attributes for a port, including extra\_dhcp\_opts extension
attributes.

Normal response Code: 200 OK

Error response Code: 401 Unauthorized.

This operation allow for the updating of attributes for the port
specified in the request URI, its port attributes, including the
extra\_dhcp\_opts attributes.

**Example Update port with extra-dhcp-opt attributes: JSON
request**

.. code::

    {
        "port":
        {
            "extra_dhcp_opts": [{"opt_value": "testfile.1", "opt_name": "bootfile-name"}]
         }
    }



**Example Update port with extra-dhcp-opt attributes: JSON
response**

.. code::

    {
        "port":
        {
            "status": "DOWN",
            "binding:host_id": null,
            "name": "",
            "allowed_address_pairs": [],
            "admin_state_up": true,
            "network_id": "87733bcc-8144-41b1-bb6b-d011d7a5168e",
            "tenant_id": "7ea98790cd854fb5a82ef3d41e5c156b",
            "extra_dhcp_opts":
            [
                {"opt_value": "123.123.123.123", "opt_name": "tftp-server"},
                {"opt_value": "testfile.1", "opt_name": "bootfile-name"},
                {"opt_value": "123.123.123.45", "opt_name": "server-ip-address"}
            ],
            "binding:vif_type": "ovs",
            "device_owner": "",
            "binding:capabilities": {"port_filter": true},
            "mac_address": "fa:16:3e:43:3c:b7",
            "fixed_ips": [{"subnet_id": "99a8aea3-b9da-409d-a5e5-f45338ceb4d3", "ip_address": "172.24.4.230"}],
            "id": "055d27c0-0194-4782-be45-275ff2c95c61",
            "security_groups": ["9bf6f19a-ba4a-470f-b8ce-28c9ad66556c"],
            "device_id": ""
        }
    }



