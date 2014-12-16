..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================
Allow the external IP address of a router to be specified
=========================================================

https://blueprints.launchpad.net/neutron/+spec/specify-router-ext-ip

There currently is no way to specify the IP address given to a
router on its external port. This blueprint allows external IPs
to be set and the action is restricted to admin-only by default.

This spec was originally approved for Juno, however due to time
constraints and conflicts with all of the DVR work ongoing at the
end of the cycle, the code was reduced to a read-only version at
the deadline.

The remaining code to finish the work is already complete and
has received several reviews.[1] It affects about 100 lines of
the L3 code so it has a small footprint and shouldn't take too
much additional effort of reviewers to merge.


Problem Description
===================

The current router API doesn't allow any control over the IP
address given to the external interface on router objects.
This makes it difficult for scenarios where tenant routers have
to be assigned a well-known address that receives special
treatment on the provider network.

Or, even if the address was originally randomly assigned,
there is no way to delete the router, move it to another project,
and preserve the previously assigned address.


Proposed Change
===============

Allow the external IP to be specified for a router in the
external_gateway_info passed to router_update. By default, this
will be restricted by policy.json to an admin-only operation.

The format of this will be the standard fixed_ips format used
when specifying an IP address for a normal port so it offers
the flexibility of specifying a subnet_id instead of an IP directly.

Requested addresses will be permitted to be any address inside any of the
subnets associated with the external network except for the gateway addresses.
They will not be affected by allocation pool ranges.

If an address is already in use, the API will return a Conflict
error (HTTP 409).

Alternatives
------------

N/A

Data Model Impact
-----------------

N/A

REST API Impact
---------------

'external_fixed_ips' is a field under 'external_gateway_info' that contains
the external IP address of the router interface. This field already exists
in the current API due to the previous partial implementation that allows
the addresses to be read. The only difference is that the field can now be
updated by an admin (or other user with the privileges defined in policy.json).

+-------------------+--------+----------+----------+------------------+--------------+
|Attribute          |Type    |Access    |Default   |Validation/       |Description   |
|Name               |        |          |Value     |Conversion        |              |
+===================+========+==========+==========+==================+==============+
|external_fixed_ips |fixed_ip|RO, owner |generated |Same as fixed_ips |External IP   |
|                   |format  |RW, admin |          |field validation  |addresses     |
|                   |for     |          |          |for normal ports. |              |
|                   |ports   |          |          |                  |              |
+-------------------+--------+----------+----------+------------------+--------------+

Right now only one fixed IP may be specified, but this may be adjusted in the
future if routers support multiple external IPs.


Security Impact
---------------

N/A if the default policy.json is left unmodified. If it's modified to allow
all users to set an IP, standard users will be allowed to ignore the allocation
ranges defined on the external subnet.

Notifications Impact
--------------------

N/A

IPv6 Impact
-----------
The IP validation will use the same validation that is used for any port IP
address so this change should be IPv6 compatible.

Other End User Impact
---------------------

N/A

Performance Impact
------------------

N/A

Other Deployer Impact
---------------------

N/A

Developer Impact
----------------

N/A

Community Impact
----------------

The community will rejoice in elation that such an amazing feature is
even possible, let alone implemented, in software.


Implementation
==============

Assignee(s)
-----------

kevinbenton

Work Items
----------

* Make the changes to the L3 db code, API, and policy.
* Update neutronclient


Dependencies
============

N/A

Testing
=======

Tempest Tests
-------------
N/A

Functional Tests
----------------
N/A

API Tests
---------

Unit tests should be adequate since there will be no new behavior outside
of the IP address assignment, which is well contained in the neutron code.


Documentation Impact
====================

User Documentation
------------------

Indicate that tenants can see their router's external IP and that
admins can specify router IPs.

Developer Documentation
-----------------------

The developer API documentation will need to be updated to indicate
that the external router IP can now be set.


References
==========

1. https://review.openstack.org/#/c/83664/

Related bugs:

https://bugs.launchpad.net/neutron/+bug/1255142

https://bugs.launchpad.net/neutron/+bug/1188427
