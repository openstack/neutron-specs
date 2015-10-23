============================
Provider networks (provider)
============================

The *``provider``* extended attributes for networks enable
administrative users to specify how network objects map to the
underlying networking infrastructure. These extended attributes also
appear when administrative users query networks.

To this aim, it extends the **network** resource by defining a set of
attributes prefixed with provider.

These attributes are added to the **network** resource:

-  provider:network\_type - Specifies the nature of the physical network
   mapped to this network resource. Examples are ``flat``, ``vlan``, or
   ``gre``.

-  provider:physical\_network - Identifies the physical network on top
   of which this network object is being implemented. The OpenStack
   Networking API does not expose any facility for retrieving the list
   of available physical networks. As an example, in the Open vSwitch
   plug-in this is a symbolic name which is then mapped to specific
   bridges on each compute host through the Open vSwitch plug-in
   configuration file.

-  provider:segmentation\_id - Identifies an isolated segment on the
   physical network; the nature of the segment depends on the
   segmentation model defined by ``network_type``. For instance, if
   ``network_type`` is ``vlan``, then this is a ``vlan`` identifier;
   otherwise, if ``network_type`` is ``gre``, then this will be a
   ``gre`` key.

The actual semantics of these attributes depend on the technology back
end of the particular plug-in. See the plug-in documentation and the
*OpenStack Cloud Administrator Guide* to understand which values should
be specific for each of these attributes when OpenStack Networking is
deployed with a particular plug-in. The examples shown in this chapter
refer to the Open vSwitch plug-in.

The default policy settings enable only users with administrative rights
to specify these parameters in requests and to see their values in
responses. By default, the provider network extension attributes are
completely hidden from regular tenants. As a rule of thumb, if these
attributes are not visible in a GET /networks/<network-id> operation,
this implies the user submitting the request is not authorized to view
or manipulate provider network attributes.
