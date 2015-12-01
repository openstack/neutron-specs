..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Network IP Availability Extension
=================================

https://bugs.launchpad.net/neutron/+bug/1457986

Problem Description
===================

**Wouldn't it be nice if we knew when a network or one of its subnets were
nearly exhausted of IPs?**

The basic need: Determining how much of a network's or subnet's IPs are being
consumed with respect to the total number of IPs they support. Ideally
this information would provide the following for networks as well as subnets:

* used_ips (number allocated or reserved within that subnet or network)
* total_ips (maximum total number available as defined by the subnet)

This blueprint describes a simple Neutron API extension that adds the
ability for an administrator (or monitor) to call an API that returns a
breakdown of IP availability details for each network. A monitor could then
alert cloud operators when thresholds have been crossed so they know to add
more capacity. The goal of this blueprint is to provide a simple API extension
that will get this IP availability information.

Use cases:

* Admin or monitor invokes API extension to get detailed information about
  network IP availability. From that data, admin or monitor can act on networks
  that are nearly full.
* Provide foundation API for a future nova scheduler to filter out or weigh
  compute hosts based on a host's backing network capacity.
* Provide counts for number of IPs used within a network or subnet.
  Answers the question: When a network is down, how many IPs are being
  affected (applies to IPv4 and IPv6)

Proposed Change
===============

We propose adding a read-only (GET) API extension that retrieves IP
availability information from existing data structures currently in OpenStack.

To create this API, we plan on creating a services extension that an admin
can configure and add to their deployment. This could also be implemented
(if community so desires) permanently into the API (non-extension).

**GET /v2.0/network-ip-availability**

* Support for filter query parameters:

  * network_id (network id as a uuid)
  * network_name (name of network)
  * ip_version (IP version either 4 or 6)
  * tenant_id (owning tenant as uuid)

* Access: admin (via json.policy)
* Read-only: **No** part of this API will change or modify data.
* Response Data:

  * Network IP availability: A network_ip_availability response that contains
    a list of networks (see network information)
  * Network information: network id, network name, tenant id, list of subnet
    sections (see subnet information), number of IPs used in network, maximum
    number of IPs that network could support.
  * Subnet information: subnet id, subnet name, IP version, cidr,
    number of IPs used in subnet, maximum number of IPs that subnet could
    support.


Proposed Change
===============

View from orbit of how things will happen for new REST API:

* API Extension created for IP availability count information
* Extension queries database to fetch Allocation, AllocationPool, Subnet,
  number of IPs used, and total counts IPs within each subnet.
* Extension generates and returns a response with a list of networks as well
  as a nested list of subnet information in each network.
* Implementation code changes should be restricted to stay within the
  extension itself and not touch other areas of Neutron.
* Python client modified to support fetching this data.
* CLI modified to support this fetching this data.
* Other modifications deemed necessary for complete end-to-end experience.

Scope for initial release
*************************

Items targeting Mitaka release (subject to review/comments/changes/help)

#. (m1) Create the above working REST endpoint.
#. (m1) Mature and flesh out unit/integration testing new extension API.
#. (m1) Update Neutron client to support this new API.
#. (m?) Design and create equivalent CLI command.
#. (m?) Add and/or modify page(s) in Horizon to display used and total counts.

Answers to questions and concerns
*********************************

The following are answers to questions and concerns received in the past:

* This extension will support IPv4 and IPv6 as well as the ability to filter
  for each.
* IPv6 total IP counts will be astronomical. This is known and is a key
  reason we plan to support ip_version as a query parameter.
* As new read-only API extension, this should not affect any known drivers,
  services, routing, DHCP, agents, scheduling, quota, etc.
* Serves up existing subnet and IP allocations information as it is in Neutron
  today (no outside data/structure/code changes needed to support this).
* IPAM concerns: IPAllocations and IPAllocationPools are maintained up to date
  on neutron side regardless of IPAM driver. Therefore IPAM driver implementers
  will be free of the need to implement an abstract function to support this.
* Floating IPs: Because we are targeting IPAllocations and pools, this spec
  will handle floating IP networks where floating IPs are reserved (created),
  but not yet associated to a fixed IP. Port counts are not able to support
  this.

References
==========

* RFE Bug: https://bugs.launchpad.net/neutron/+bug/1457986
* Proof of concept code: https://review.openstack.org/#/c/212955
