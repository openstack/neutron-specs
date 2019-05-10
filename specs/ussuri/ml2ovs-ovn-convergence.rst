..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Toward Convergence of ML2+OVS+DVR and OVN
=========================================

Launchpad Bug:
https://bugs.launchpad.net/neutron/+bug/1828607

In its current state, DVR lacks some functionality that would be of benefit to
operators. When looking at ways to address these items, it has become
apparent there is substantial overlap in functionality between the desired
state of DVR on ML2+OVS and OVN. This spec is meant to outline the areas of
overlap and the areas where either OVN or ML2+OVS+DVR are lacking
functionality, then describe a plan for converging these 2 solutions.


Problem Description
===================

In its current state, ML2+OVS+DVR has the following gaps:

- Control plane scalability when handling both large numbers of ports and
  when large numbers of nodes participate in the neutron deployment.
- Support for distributed ingress/egress when using IPv6 in conjunction with
  BGP and/or neutron-dynamic-routing.
- DVR cannot be cleanly offloaded with smart-NIC technology or user-space
  tools due to its use of linux namespaces.

In addition to the above shortcomings, DVR would also benefit from the
following enhancements:

- Support for running without network nodes if an operator chooses
- Support for distributed DHCP (local DHCP responder) [1]_
- Support for distributed SNAT

When looking at enhancing control plane scalability, distributed DHCP, and
pushing all forwarding policy into OVS for potential hardware offload, it
has been noted that OVN already implements many of these things.  OVN also
lacks some functionality that ML2+OVS+DVR has such as fully-distributed ingress
and egress which makes it possible to route all north-south traffic directly
to the appropriate compute, bypassing a central network node.

Proposed Change
===============

To address the issues previously discussed, the ML2+OVS+DVR solution will be
merged with the networking-ovn solution. As a desired end state, neutron will
have OVN as its reference ML2 plugin for OVS-based environments. This has the
following benefits:

- Neutron can take advantage of the control plane performance optimizations
  OVN provides.
- Removes the duplication of effort for both core neutron developers and
  networking-ovn developers, and allows for cross-pollination of ideas for
  addressing scale, data plane performance, and usability issues in neutron.
- Aligns the community of developers supporting neutron and OVN, broadening
  the potential contributor base for neutron.

For this effort to be successful and minimally disruptive to users, the
following things should be considered:

- Will users actually observe better control plane scale?
- Will users lose support for any use cases?
- What is the migration path? How disruptive will migration be? How can it be
  simplified?

The following sections attempt to answer these questions and outline the
procedural and technical aspects of the convergence of ML2+OVS+DVR and
networking-ovn.

Feature Gap Analysis
--------------------

This is a list of some of the currently known gaps between ML2/OVS and OVN.
It is not a complete list, but is enough to be used as a starting point for
implementors working on closing these gaps. A TODO list for OVN is located
at [2]_.

- Fragmentation and Jumbo frames support

  * Upstream work to support this in the OVS/OVN code has been completed,
    additional work is required in the networking-ovn repository to enable it.
    Change at [3]_.

- Port forwarding

  * Currently ML2/OVS supports Port Forwarding in the North/South plane.
    Specific L4 Ports of the Floating IP can be directed to a specific
    FixedIP:PortNumber of a VM, so that different services running in a VM
    can be isolated, and can communicate with external networks easily.

  * This is a relatively new extension, support would need to be added to OVN.

    One possible way would be to use the OVN native load balancing feature.
    An OVN load balancer is expressed in the OVN northbound load_balancer
    table. Normally the VIP and its members are expressed as [4]_:

    VIP:PORT = MEMBER1:MPORT1, MEMBER2:MPORT2

    The same could be extended for port forwarding as:

    FIP:PORT = PRIVATE_IP:PRIV_PORT

- Security Groups logging API

  * ML2/OVS supports a log file where security groups events are logged to be
    consumed by a security entity. This allows users to have a way to check if
    an instance is trying to execute restricted operations, or access
    restricted ports in remote servers.

  * This is a relatively new extension, support would need to be added to OVN.

- QoS DSCP support

  * Currently ML2/OVS supports QoS DSCP tagging and egress bandwidth limiting.
    Those are basic QoS features that while integrated in the OVS/OVN C core
    are not integrated (or fully tested) in the neutron OVN mechanism driver.

- QoS for Layer 3 IPs

  * Currently ML2/OVS supports floating IP and gateway IP bandwidth limiting
    based on Linux TC. Networking-ovn has a prototype implementation [5]_ based
    on the meter of openvswitch [6]_ utility, which is supported in user space
    datapath only, or kernel versions 4.15+ [7]_.

- QoS Minimum Bandwidth support

  * Currently ML2/OVS supports QoS Minimum Bandwidth limiting, but it is
    not supported in OVN.

- BGP support

  * Currently ML2/OVS supports making a tenant subnet routable via BGP, and
    can announce host routes for both floating and fixed IP addresses.

Performance Impact
------------------

Performance tests carried out in late 2018 showed OVN outperforming ML2/OVS
in most operations [8]_. Only creating networks and listing ports were
slower, which is mostly due to the fact that OVN creates an extra port
(for metadata) upon network creation, so the amount of ports listed for
the same rally task is 2x for the OVN case.

Also, the resources utilization is lower in OVN [9]_ vs ML2/OVS
[10]_ mainly due to the lack of agents and from not using RPC.

One question that has been raised is OVNs use of Geneve instead of VXLAN,
and if that introduces any performance issues. Research showed that this
is not a problem. Normally, NICs that support VXLAN also support
Geneve hardware offload. Interestingly, even in the cases where they
do not, performance was found to be better using Geneve due to other
optimizations that Geneve benefits from. More information can be found
in Russell Bryant's blog [11]_, who did extensive work in this space.

Other Deployer Impact
---------------------

Migration from ML2/OVS to OVN is specific to the deployment tool being used.
Currently there is documentation on migration using TripleO at [12]_, some
of which might be applicable to other deployment tools. Re-writing the
information into different sections - one being a generic, high-level
description of what steps are required, and the other being deployment-specific
notes, would help others when adding more tools.

Developer Impact
----------------

- Move networking-ovn code to neutron repo and treat OVN as an in-tree driver

  * Moving networking-ovn into the neutron repository is the first step toward
    supporting convergence.

  * An etherpad to track information and work items is located at [13]_.

- Distributed ingress/egress for IPv6

  * In its current state, DVR supports distributed ingress and egress when
    using IPv4. When a floating IP is in use, it can either be routed with a
    /32 host route directly to the fip-xxxxx namespace on the compute node
    (when using neutron-dynamic-routing) or gratutitous ARP can be sent from
    the fip-xxxxx namespace for the floating IP.
  * When IPv6 is used, all cloud ingress and egress traffic is sent through the
    centralized router. This means that traffic cannot be steered directly to
    compute nodes.
  * To address this, distributed ingress/egress (AKA "fast-exit") would be
    implemented for IPv6. This will allow IPv6 packets arriving in the
    fip-xxxxx namespace to be routed through the appropriate qrouter-xxxxx
    namespace on the compute node, bypassing the centralized router hosted on
    a network node.
  * There is an existing RFE for this, see [14]_.

- Support for smart-NIC or user-space offloads

  * This involves pushing all DVR forwarding policy into OVS and implementing
    it via OpenFlow.
  * As all forwarding/security policy is pushed directly to OVS, an evaluation
    of what flow rules are currently being offloaded to hardware should be
    done if it is possible. If possible, any rules that should be tweaked for
    optimizing offload potential should be re-written to support offloading.
  * There is an existing blueprint for openflow-based DVR in neutron, see [15]_,
    but the proof of concept patch has been abandoned [16]_.

- Allow for completely de-centralized operation

  * This involves making it possible for operators to run their environments
    without introducing the concept of network nodes.
  * This implies support for distributed SNAT, distributed DHCP, and
    distributed ingress/egress for IPv4 and IPv6.
  * Compatibility with neutron-dynamic-routing for routing directly to tenant
    networks [17]_.
  * In a completely de-centralized deployment, a router as a logical construct
    doesn't need to be thought of as "HA" or "distributed" as it is inherently
    both by virtue of being instantiated on each compute node.

- Distributed SNAT

  * This involves allowing SNAT to happen directly on the compute node instead
    of centralizing it on a network node.
  * To provide proper tenant isolation when using the namespace-based
    implementation of DVR, SNAT would be performed twice
    (Insert diagram and explain in detail here).
  * Open questions:

    * Should SNAT traffic use the FIP gateway IP as the source address, or
      should it use an IP address specific to the tenant router? Using the
      FIP gateway IP seems simpler and does cut down on address usage
      (although subnet service types does make this less of an issue). On the
      flip side, an address associated with the router does allow for better
      scrutiny of network traffic as traffic flows can be tied to a specific
      project and router.
    * Allowing operators to choose whether to use distributed SNAT or not
      is the best way forward.
  * See [18]_ for some historical context and details.

CLI Impact
----------
No CLI impact is anticipated. All existing CLI commands will continue to be
supported.

API Impact and Required Extensions
----------------------------------
The extensions supported by OVN plugins vs. ML2+OVS should provide equivalent
functionality in terms of data plane features such trunks, routed networks,
address scopes, etc. Matching support for every specific extension is not
necessary in all cases. For example, OVN provides DVR functionality in its
implementation without implementing the API extensions DVR introduces.

Implementation
==============

Assignee(s)
-----------

* Brian Haley <haleyb.dev@gmail.com>

Work Items
----------

* Move networking-ovn into the neutron tree.
* Ensure RFE's have been created for all items in feature gaps and developer
  impact sections.
* Rally jobs for capturing control plane performance with OVN in place.

References
==========

.. [1] https://blogs.rdoproject.org/2016/08/native-dhcp-support-in-ovn/
.. [2] https://github.com/ovn-org/ovn/blob/master/TODO.rst
.. [3] https://review.opendev.org/#/c/671766/
.. [4] https://github.com/ovn-org/ovn/blob/master/ovn-nb.ovsschema#L148
.. [5] https://review.opendev.org/#/c/539826/
.. [6] https://github.com/openvswitch/ovs/commit/66d89287269ca7e2f7593af0920e910d7f9bcc38
.. [7] https://github.com/torvalds/linux/blob/master/net/openvswitch/meter.h
.. [8] https://imgur.com/a/4QtaN6b
.. [9] https://imgur.com/a/N9jrIXV
.. [10] https://imgur.com/a/oOmuAqj
.. [11] https://blog.russellbryant.net/2017/05/30/ovn-geneve-vs-vxlan-does-it-matter/
.. [12] https://docs.openstack.org/networking-ovn/latest/install/migration.html
.. [13] https://etherpad.openstack.org/p/ML2-OVS-OVN-Convergence
.. [14] https://bugs.launchpad.net/neutron/+bug/1774463
.. [15] https://blueprints.launchpad.net/neutron/+spec/openflow-based-dvr
.. [16] https://review.opendev.org/#/c/472289/
.. [17] https://docs.openstack.org/neutron/latest/admin/config-bgp-dynamic-routing.html
.. [18] https://etherpad.openstack.org/p/boston-dvr
