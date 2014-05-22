..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================
Allow the external IP address of a router to be specified
=========================================================

https://blueprints.launchpad.net/neutron/+spec/specify-router-ext-ip

There currently is no way to specify the IP address given to a
router on its external port. The API also does not return the
IP address that is assigned to the router. This is a problem
if the router is running a service (e.g. VPNaaS) that requires
the address to be known. This blueprint allows external IPs to
be set (admin-only by default) and allows the IP to be read.




Problem description
===================

The current router API doesn't allow any control over the IP
address given to the external interface on router objects. It
also blocks tenants from reading the IP address it is given.
This makes it difficult for tenants in scenarios where the IP
address needs to be known or needs to be set to a known address.

For example, if the router is running VPNaaS, the tenant can't
get the address required to issue to clients. Or, even if the address
is already known to clients, there is no way to delete the router,
move it to another project, and request the same address.


Proposed change
===============

Allow the external IP to be specified for a router in the
external_gateway_info passed to router_update. By default, this
will be restricted by policy.json to an admin-only operation.

Include the external IP addresses in the get_router response so
tenants can see the addresses.

The format of this will be the standard fixed_ips format used
when specifying an IP address for a normal port so it offers
the flexibility of specifying a subnet_id instead of an IP directly.

The fixed_ips format will also handle use cases where the router
has multiple external addresses. For example, a logical
router may be implemented in a distributed fashion with multiple external
addresses to distribute the source NAT traffic. In the current reference
implementation, it will just raise an exception if a user tries to add
more than one.

Requested addresses will be permitted to be any address inside any of the
subnets associated with the external network except for the gateway addresses.
They will not be affected by allocation pool ranges.

If an address is already in use, the API will return a BadRequest
error (HTTP 400).

Alternatives
------------

N/A

Data model impact
-----------------

N/A

REST API impact
---------------

Adds a new external_ips field to the external_gateway_info dict
in the router object.

+-------------------+--------+----------+----------+------------------+--------------+
|Attribute          |Type    |Access    |Default   |Validation/       |Description   |
|Name               |        |          |Value     |Conversion        |              |
+===================+========+==========+==========+==================+==============+
|external_fixed_ips |fixed_ip|RO, owner |generated |Same as fixed_ips |External IP   |
|                   |format  |RW, admin |          |field validation  |addresses     |
|                   |for     |          |          |for normal ports. |              |
|                   |ports   |          |          |                  |              |
+-------------------+--------+----------+----------+------------------+--------------+



Security impact
---------------

N/A

Notifications impact
--------------------

N/A

Other end user impact
---------------------

N/A

Performance Impact
------------------

N/A

Other deployer impact
---------------------

N/A

Developer impact
----------------

N/A

Implementation
==============

Assignee(s)
-----------

kevinbenton

Work Items
----------

Make the changes to the L3 db code, API, and policy.
Update neutronclient

Dependencies
============

N/A

Testing
=======

Unit tests should be adequate since there will be no new behavior outside
of the IP address assignment, which is well contained in the neutron code.


Documentation Impact
====================

Indicate that tenants can see their router's external IP and that
admins can specify router IPs.


References
==========

https://bugs.launchpad.net/neutron/+bug/1255142

https://bugs.launchpad.net/neutron/+bug/1188427
