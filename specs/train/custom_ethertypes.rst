..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
API Support for Managing Custom Ethertypes
==========================================

Some operators need to allow/deny custom Ethertypes for applications which use
their own non-IP traffic (such as for clustering applications). The Security
Group API only handles specifying behavior within the IP protocol. With the
firewall reference implementation (OVS Firewall) anything other than IPv4 and
IPv6 is subject to the default deny. This means OpenStack customers have no
options to use OpenStack to permit protocols that use separate ethertypes like
InfiniBand and FCoE.

This specification proposes adding to the Security Group API the capability to
specify standard security group behaviors (allow, deny) for custom ethertypes,
with the aim of implementing these controls in the OVS and OVN firewalls.



Problem Description
===================

The first implementation of security groups in Openstack was based on iptables,
which only controls traffic for the IPv4 or IPv6 ethertypes.  Since all
subsequent implementations have been built on the conceptual foundation of that
initial implementation, the behavior of Neutron security groups for traffic that
has other ethertypes has been undefined.

In the case of the OVS Firewall driver, the behavior is for traffic with non-IP
ethertypes to fall through to the default deny action.  A flow could be added by
an operator to permit this traffic, but it would be removed the next time the
neutron-openvswitch-agent restarted.  This is a runtime requirement, so
deployment time tools like TripleO or Ansible are unable to handle it.

Use case: Customer is running an application using InfiniBand (ethertype 0x4008)
in OpenStack, and that OpenStack transitions from iptables_hybrid to OVS
firewall. The Infiniband traffic is blocked by the OVS firewall, and at present
the Neutron API offers no methodology to unblock it.



Proposed Change
===============

The best solution is a new API extension that makes Neutron aware of permitted
ethertypes and adjust OVS firewall rules appropriately.  The current security
group APIs take an ethertype option, which is constrained to IPv4 and IPv6;
implementing this spec will lift that restriction, so that additional security
types can be specified by hexadecimal number.  If a non-IP ethertype is
specified then the following arguments are ignored: protocol, port_range_max,
port_range_min, remote_group_id, remote_ip_prefix.

Control for traffic with a custom ethertype will be all or nothing, a given
protocol will be either entirely blocked or entirely allowed.

The 'ethertype' field of a SecurityGroupRule object is of type EtherTypeEnumField
which is an enum of the VALID_ETHERTYPES constant; this would need to be altered
to be a two octet hexadecimal number between 0x0000 and 0xffff.  The security
group rule database field for ethertypes is a String(40), so no modification is
needed.  A validator validate_ethertype is suggested, which would also replace
use of sg_supported_ethertypes in the security group extension.

References
==========

* RFE bug report of this spec: https://bugs.launchpad.net/neutron/+bug/1832758

