..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
IPv6 support in Metadata service
================================

https://bugs.launchpad.net/neutron/+bug/1460177

Adding IPv6 support for Metadata service.

Problem Description
===================

The metadata service uses the well-known IP 169.254.169.254 as its endpoint
address. This doesn't work in an IPv6-only environment.

169.254.169.254 is not accessible from an IPv6-only Virtual Machine (VM).
It is possible to add 169.254.0.0/16 to a port's allowed_address_pairs list
and use an IPv6 link-local address for interface configuration, but the
metadata proxy doesn't even start if there are no DHCP-enabled IPv4 subnets,
and IPv6 link-local addresses are unknown for Neutron and can't be used for
instance identification.

Proposed Change
===============

The metadata proxy starts unconditionally when a network is assigned to a DHCP
agent or L3 router.

The metadata proxy listens on a dual-stack socket (::).

In the case of IPv4, the metadata proxy uses IP address 169.254.169.254 which
belongs to the IPv4 link-local subnet (169.254.0.0/16 according to [1]).
In the case of IPv6, the metadata proxy joins the anycast group fe80::a9fe:a9fe
on all available interfaces.
The fe80::a9fe:a9fe IP address is equivalent of 169.254.169.254 in the IPv6
link-local subnet which is fe80::/10 according to [2].
It is valid to do this because it is running inside a router or dhcp namespace.

The L3 agent will add a firewall rule that redirects traffic sent to
the proposed anycast IP (fe80::a9fe:a9fe) to the metadata proxy port.
In the case of the DHCP agent, a new IP address (fe80::a9fe:a9fe) will be
configured on the tap port which belongs to the DHCP agent, the same way it is
currently done for the IPv4 address (169.254.169.254).

The VM uses the address fe80::a9fe:a9fe to access Metadata service. Software
like cloud-init used inside VMs must be aware of this new IPv6 address. So
images used in Openstack clouds will have to be updated as well.

When the metadata proxy processes a request, it gathers the L2 addresses of a
VM, and the source interface, and passes it to the metadata service.

The Metadata service, instead of using the VM IP, uses the "VM MAC" and
"Gateway MAC" to identify the instance.

Documentation Impact
====================

As this new IPv6 address proposed for metadata service isn't currently used by
any other cloud provider, we need to update our documentation to make it very
clear what IPv6 address is used by the metadata service and how to configure
it in at least the most popular metadata consumer, which is ``cloud-init``.
In the case of ``cloud-init``, this new IP address can be set using the
``metadata_urls`` config option [3].

In the future we can update the ``cloud-init`` code to make this IPv6 address
be one of the default IPs it uses for the OpenStack Datasource, or propose a
new datasource, for example, OpenStackIPv6.

References
==========

[1] https://tools.ietf.org/html/rfc3927
[2] https://tools.ietf.org/html/rfc4291
[3] https://github.com/canonical/cloud-init/blob/4f940bd1f76f50f947af533661ba6fafa3e60e59/doc/rtd/topics/datasources/openstack.rst
