===========================
Layer-3 networking (router)
===========================

The Layer-3 networking extension enables OpenStack Networking API users
to route packets between subnets, forward packets from internal networks
to external ones, and access instances from external networks through
floating IPs.

The OpenStack Networking Layer-3 extension defines these resources:

-  **router**. A logical entity that forwards packets across internal
   subnets and NATs them on external networks through an appropriate
   external gateway. A router has an interface for each subnet with which it is
   associated. By default, the IP address of such interface is the
   subnet's gateway IP. Also, whenever a router is associated with a
   subnet, a port for that router interface is added to the subnet's
   network.

-  **floating IP**. Represents an external IP address that is mapped to
   an OpenStack Networking port and, optionally, a specific IP address
   on a private OpenStack Networking network. A floating IP enables
   access to an instance on a private network from an external network.
   Floating IPs can only be defined on networks where the
   router:external attribute (by the external network extension) is set
   to ``True``.

.. csv-table:: Router attributes
   :header: "Attribute", "Type", "Required", "CRUD", "Default value", "Validation constraints", "Notes"
   :widths: 18, 4, 3, 4, 4, 4, 30

   "id", "uuid-str", "N/A", "R", "generated", "N/A", "Unique identifier for the router."
   "name", "String", "No", "CRU", "None", "N/A", "Human readable name for the router. Does not have to be unique."
   "admin\_state\_up", "Bool", "No", "CRU", "true", "{true\false}", "Administrative state of the router."
   "status", "String", "N/A", "R", "N/A", "N/A", "Indicates whether or not a router is currently operational."
   "tenant\_id", "uuid-str", "No", "CR", "Derived from Authentication token", "N/A", "Owner of the router. Only admin users can specify a tenant identifier
   other than its own."
   "external\_gateway\_info", "dict", "No", "CRU", "None", "No constraint", "Information on external gateway for the router."

.. csv-table:: Floating IP attributes
   :header: "Attribute", "Type", "Required", "CRUD", "Default value", "Validation constraints", "Notes"
   :widths: 18, 4, 3, 4, 4, 4, 30

   "id", "uuid-str", "N/A", "R", "generated", "N/A", "Unique identifier for the floating IP instance."
   "floating_network_id", "uuid-str", "Yes", "CR", "N/A", "UUID Pattern", "UUID of the external network where the floating IP is to be created."
   "port_id", "uuid-str", "Yes", "CRU", "N/A", "UUID Pattern", "UUID of the port on an internal OpenStack Networking network which should be associated to the floating IP."
   "fixed_ip_address", "IP Address", "No", "CRU", "None", "IP address or null", "Specific IP address on port_id which should be associated with the floating IP."
   "floating_ip_address", "IP Address", "N/A", "R", "Address of the floating IP on the external network."
   "tenant\_id", "uuid-str", "No", "CR", "Derived from Authentication token", "N/A", "Owner of the floating IP. Only admin users can specify a tenant identifier other than its own."

..note:
    C means, use the attribute in create operations. R means this attribute is
    returned in response to show and list operations. U means you can update the
    value of this attribute. D means you can delete the value of this attribute.
