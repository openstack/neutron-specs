=========================================
Ports binding extended attributes (ports)
=========================================

Use the Networking API v2.0 with the *``binding``* extended attributes
to get information about, create, and update port objects.

The *``binding``*-prefixed extended attributes for ports are:

**Table Ports binding extended attributes**

Attribute

Type

Required

CRUD\ `:sup:`[a]` <#ftn.crud_network>`__

Default value

Validation constraints

Notes

*``binding:vnic_type``*

String

N/A

CRU

normal

(normal, direct, macvtap)

The vnic type to be bound on the neutron port.

In **POST** and **PUT** operations, specify a value of ``normal``
(virtual nic), ``direct`` (pci passthrough), or ``macvtap`` (virtual
interface with a tap-like software interface). These values support
SR-IOV PCI passthrough networking. The ML2 plug-in supports the
*``vnic_type``*.

In **GET** operations, the *``binding:vnic_type``* extended attribute is
visible to only port owners and administrative users.

*``binding:vif_type``*

String

N/A

R

None

N/A

Read-only. The vif type for the specified port.

Visible to only administrative users.

*``binding:vif_details``*

list(dict)

N/A

R

None

N/A

Read-only. A dictionary that enables the application to pass information
about functions that Networking API v2.0 provides. Specify the following
value: ``port_filter :                             Boolean`` to define
whether Networking API v2.0 provides port filtering features such as
security group and anti-MAC/IP spoofing.

Visible to only administrative users.

*``binding:host_id``*

uuid-str

N/A

CRU

None

N/A

The ID of the host where the port is allocated. In some cases, different
implementations can run on different hosts.

Visible to only administrative users.

*``binding:profile``*

list(dict)

N/A

CRU

None

N/A

A dictionary that enables the application to pass information about
functions that the Networking API provides. To enable or disable port
filtering features such as security group and anti-MAC/IP spoofing,
specify ``port_filter: True`` or ``port_filter: False``.

Visible to only administrative users.

-  **`:sup:`[a]` <#crud_network>`__\ C**. Use the attribute in create
   operations.

-  **R**. This attribute is returned in response to show and list
   operations.

-  **U**. You can update the value of this attribute.

-  **D**. You can delete the value of this attribute.


