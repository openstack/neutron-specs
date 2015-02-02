..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
add-port-timestamp
==========================================

https://blueprints.launchpad.net/neutron/+spec/add-port-timestamp

It is worth adding timestamp fields including create_at, update_at in the table
of port in neutron, so users can monitor port change conveniently, for example
portal or management center would like to query the ports that have changed or
refreshed during the latest 5 seconds in a large scale application. If not,
it's time-consuming and inefficient.


Problem Description
===================

Currently we don't have timestamp fields associated with port in neutron, so
there's no way that we query ports based on the time they are operated. End
users may be interested in ports operated in a specific period for monitoring
or statistics purpose but this requirement can not be fulfilled currently.


Proposed Change
===============

We propose to add create_at and update_at fields to port model. Both the
default value of create_at and update_at are set to timeutils.utcnow(). Also
onupdate trigger is set to update_at to refresh update_at to timeutils.utcnow()
when a port is updated. All the timestamp field operation can be finished in
the database side, so there is no extra work needed in the plugin side.

Based on these fields, change_since filter is introduced when retrieving ports.
It accepts a timestamp and neutron will return ports whose update_at fields are
later than this timestamp.

Data Model Impact
-----------------

Two new attributes added to ports:

+----------------+----------+--------------------+----------+------+---------------------------+
| Attribute name | Type     | Default Value      | Required | CRUD | Description               |
+----------------+----------+--------------------+----------+------+---------------------------+
| created_at     | DateTime | timeutils.utcnow() | Y        |      | Timestamp port is created |
| updated_at     | DateTime | timeutils.utcnow() | Y        |      | Timestamp port is updated |
+----------------+----------+--------------------+----------+------+---------------------------+

Both created_at and updated_at are maintained by neutron and users do not care
about them, so CRUD are all empty.

REST API Impact
---------------

New added field create_at and updated_at are maintained by neutron so they will
not bring API impact.

Ports list API will accept new query string parameter change_since. Users can
pass time to the list API url to retrieve ports operated since a specific time.

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Python neutron client may add help to inform users this new filter. Python
neutron client support dynamic assigning search fields so it is easy for it to
support this new filter.

Performance Impact
------------------

Performance of syncing port status with other systems, like a monitor system or
a user interface, can be improved. Instead of retrieving all the ports in every
syncing period to update port status, these systems can just retrieve ports
updated during the syncing interval using the new change_since filter.

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Community Impact
----------------

This change will bring facility to monitor neutron port resources. Actually
most projects in OpenStack like nova, cinder have timestamp fields to track the
operation time of resources.

Alternatives
------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  caizhiyuan1@huawei.com

Other contributors:
  TBD

Work Items
----------

* Update database schema
* Add API filter
* Add related test

Dependencies
============

None


Testing
=======

Tempest Tests
-------------

None

Functional Tests
----------------

* Test if create_at and update_at can be correctly initiated.
* Test if update_at can be correctly written when port updated.
* Test if change_since filter can be correctly applied.

API Tests
---------

Test if the new filter can be correctly parsed and validated.


Documentation Impact
====================

User Documentation
------------------

Update neutron API reference

Developer Documentation
-----------------------

None

References
==========

None
