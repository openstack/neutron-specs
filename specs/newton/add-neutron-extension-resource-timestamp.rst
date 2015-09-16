..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
add-neutron-extension-resource-timestamp
==========================================

https://blueprints.launchpad.net/neutron/+spec/add-neutron-extension-resource-timestamp

Neutron extension resource models like router, floatingip, securitygroup and
securitygroup rules should include timestamp fields "created_at" and
"updated_at". The fields can make the monitoring changed resource task easier.


Problem Description
===================

Users may be interested in monitoring Neutron extension resources or collecting
statistics on a group of resources in a specific time-frame, for instance,
querying routers that have changed or refreshed in the last several seconds.
In Neutron, we do not have timestamp fields to support this requirement.

For another example, when a router interface had been updated, the notify
info should be sent to l3 agent, but it cannot know whether the router had
been updated if the message lost. In Neutron, there is no way to vertify
the info had been arrived or lost.


Proposed Change
===============

We propose to add created_at and updated_at fields to resource standardattr
models. In the sqlalchemy model definition, "default" argument is set to
timeutils.utcnow() for created_at and updated_at, so when a resource is
created, sqlalchemy will automatically fill these two fields with the timestamp
the resource is created; "onupdate" argument is also set to timeutils.utcnow()
for updated_at so when a resource is updated, updated_at will be refreshed.
All the timestamp field operations are finished in the database side, so there
is no extra work needed in the plugin side. If the way below is not work well,
we can add the timestamp fields to sqlalchemy model when the creation of
extension resources, and the same as the updated_at.

Based on these two fields, changed_since filter is introduced for retrieving
resources. It accepts a timestamp and Neutron will return resources with
updated_at field later than this timestamp.

Data Model Impact
-----------------

Currently, HasStandardAttributes model had contained created_at and updated_at.
We could reuse the timestamp_core code to implement. This change won't impact
on other models which contain standard attributes.

+----------------+----------+--------------------+----------+------+-------------------------------+
| Attribute name | Type     | Default Value      | Required | CRUD | Description                   |
+----------------+----------+--------------------+----------+------+-------------------------------+
| created_at     | DateTime | timeutils.utcnow() | Y        | R    | Timestamp resource is created |
| updated_at     | DateTime | timeutils.utcnow() | Y        | R    | Timestamp resource is updated |
+----------------+----------+--------------------+----------+------+-------------------------------+

As discussed above, these two fields are maintained by sqlalchemy in Neutron
server. It's not necessary for users to pass these two values when creating or
updating resource, so only Read operation in CRUD is provided.

REST API Impact
---------------

Created_at and updated_at will be returned when users issue resource retrieving
requests.

Resources list API will accept new query string parameter changed_since. Users
can pass timestamp of ISO 8601 format to the list API uri to retrieve resources
operated since a specific time.

Take router as an example, the request uri looks like this:

.. code::

  GET /v2.0/routers?change_since=2015-07-31T00:00:00

and response:

::

  {
      "router": {
          "status": "ACTIVE",
          "external_gateway_info": {
              "network_id": "85d76829-6415-48ff-9c63-5c5ca8c61ac6",
              "external_fixed_ips": [
                  {
                      "subnet_id": "255.255.255.0",
                      "ip": "192.168.10.2"
                  }
              ]
          },
          "name": "router1",
          "admin_state_up": true,
          "tenant_id": "d6554fe62e2f41efbb6e026fad5c1542",
          "routes": [],
          "id": "a07eea83-7710-4860-931b-5fe220fae533",
          "created_at": "2015-07-31T11:50:05",
          "updated_at": "2015-07-31T11:51:12",
      }
  }

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Neutron python client may add help to inform users the new filter. Neutron
python client supports dynamic assigning search fields so it is easy for it to
support this new filter. Also Neutron python client needs to add the two new
fields when displaying resource information.

Performance Impact
------------------

Performance of syncing resource status with other systems like a monitor system
or a user interface can be improved. Instead of retrieving all the resources in
every syncing period to update resource status, these systems can just retrieve
resources updated during the syncing interval using changed_since filter.

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

NTP service needs to be configured and started to synchronize time among nodes,
so the timestamps saved in created_at and updated_at are valid across nodes.

Developer Impact
----------------

None

Community Impact
----------------

This change will bring facility to monitor Neutron resources. Actually most
projects in OpenStack like `Nova`_ have timestamp fields to track the operation
time of resources.

One problem of absolute timestamp is that sudden system time change caused by
attack or failure will make the previous cached timestamp invalid. Seeking a
relative timestamp storing strategy may be a better choice, but it's out of
the extent of this blueprint.

Alternatives
------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  zhaobo6@huawei.com


Work Items
----------

* Update database schema
* Add API filter
* Add related test
* Update neutron client to support the new filter


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

* Test if created_at and updated_at can be correctly initiated.
* Test if updated_at can be correctly written when resource updated.
* Test if changed_since filter can be correctly applied.

API Tests
---------

Test if the new filter can be correctly parsed and validated.


Documentation Impact
====================

User Documentation
------------------

Update Neutron API reference.

Developer Documentation
-----------------------

Update developer documentation to introduce the new filter.


References
==========

.. target-notes::

.. _`Nova`: https://github.com/openstack/nova/blob/master/nova/db/sqlalchemy/models.py#L43

