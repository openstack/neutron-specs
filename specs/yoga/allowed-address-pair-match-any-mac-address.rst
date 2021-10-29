..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Allowed Address Pair: support matching ANY MAC address
======================================================

Include the URL of your launchpad RFE:

https://bugs.launchpad.net/neutron/+bug/1946251

Allowed address pairs API allows to match against a subset of IP addresses, or
even all of them, by virtue of netmasks being part of 'ip_address' field
format. For example, using '0.0.0.0/0' means allowing any IP address used with
the corresponding MAC address. This proposal suggests extending the API so that
a user can allow any MAC address used with the corresponding IP address.


Problem Description
===================

An end user would like to configure their topology so that a port enforces
security group rules but allows any MAC addresses. To achieve this, the user:

#. creates a security group with appropriate rules;
#. creates a port and assigns it to the security group;
#. adds allowed address pair for ANY MAC address and some IP address(es).

Right now, the only way to disable MAC address enforcement is to set
port_security_enabled attribute to false. But this attribute also disables
security group rule enforcement. In this scenario, only anti-spoofing
enforcement should be disabled.


Proposed Change
===============

The proposal is to extend 'allowed_address_pairs' attribute for ports so that
its 'mac_address' field accepts a special value "ANY" that would disable MAC
anti-spoofing mechanism. IP address enforcement may still continue, based on
the corresponding 'ip_address' field value of the pair.

Having multiple address pairs with "ANY" 'mac_address' field value is
supported.  Having a list of pairs where some but not all 'mac_address' field
values are "ANY" is supported.

A new Networking API extension will be introduced to indicate support for the
new special value. This support is backend specific, which means that ML2 may
need to offload decision to enable the extension to underlying drivers.

Database model for allowed address pairs will not need adjustment to accept the
new special value because it already accepts all strings.

Client library may need to be adjusted to support the new special value "ANY"
for the 'mac_address' field in the API validator module.

For agent-based setups, there are two cases to handle where the firewall driver
chosen implements anti-spoofing / allowed_address_pair handling itself, and
where this task is left to the agent. The firewall driver indicates the former
by setting provides_arp_spoofing_protection attribute to True. In this
scenario, the driver will have to implement handling of the new "ANY" value set
for a MAC-IP pair. A new firewall driver flag (support_any_mac_anti_spoofing)
will be added to indicate support for the new feature, defaults to False. If
the flag is not set, the agent will filter out ANY address pairs before passing
them to the firewall driver. It will also log a warning about the issue. Each
driver that handles anti-spoofing itself will need to opt in receiving "ANY"
address pairs by setting the support_any_mac_anti_spoofing flag to True.


Alternatives
~~~~~~~~~~~~

Instead of using a new special value "ANY", we could introduce a masking scheme
for 'mac_address' field. For example, "de:ad:/32" would indicate "any MAC
address that starts with de:ad: prefix". But it's debatable whether using any
mask but /0 is generally useful, since MAC addresses are opaque values with no
inherent ordering or grouping (that said, traditionally vendors reuse the same
prefixes for their devices).

Instead of reusing the existing allowed address pairs API attribute, we could
introduce a new port level attribute that would disable MAC address
anti-spoofing without disabling security group rules. Since SG API still allows
to match against particular IP addresses, this approach would be functionally
identical to this. That said, I think it's generally better to avoid
introducing new attributes to base resources, and it's better to keep
anti-spoofing related functionality under the same attribute.


References
==========
