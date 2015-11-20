..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
add-resource-timestamp
==========================================

https://blueprints.launchpad.net/Neutron/+spec/add-port-timestamp

Resource models like network, subnet and port in Neutron should include
timestamp fields "created_at" and "updated_at". These two fields can
facilitate the task of monitoring resource changes, which are time-consuming
and inefficient currently.


Problem Description
===================

End users may be interested in monitoring Neutron resources or collecting
statistics on a group of resources in a specific time-frame, for instance,
querying ports that have changed or refreshed in the last 5 seconds in a
large scale application. In Neutron, we do not have timestamp fields
associated with resources, so there is no way we can query them based on
their operational time.


Proposed Change
===============

We propose to add created_at and updated_at fields to resource models. In the
sqlalchemy model definition, "default" argument is set to timeutils.utcnow()
for created_at and updated_at, so when a resource is created, sqlalchemy will
automatically fill these two fields with the timestamp the resource is created;
"onupdate" argument is also set to timeutils.utcnow() for updated_at so when a
resource is updated, updated_at will be refreshed. All the timestamp field
operations are finished in the database side, so there is no extra work needed
in the plugin side.

Based on these two fields, change_since filter is introduced for retrieving
resources. It accepts a timestamp and Neutron will return resources with
updated_at field later than this timestamp.

Data Model Impact
-----------------

Two new attributes are added to resource model:

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

Created_at and updated_at will be exposed to users via extension API. If plugin
supports, they are returned when users issue resource retrieving requests.
Also, resources list API accepts new query string parameter change_since. Users
can pass timestamp of ISO 8601 format to the list API uri to retrieve resources
operated since a specific time.

Take port as an example, the request uri looks like this:

.. code::

  GET /v2.0/ports?changed_since=2015-07-31T00:00:00

and response:

.. code::

  {
      "ports": [
          {
              "status": "ACTIVE",
              "name": "",
              "allowed_address_pairs": [],
              "admin_state_up": true,
              "network_id": "70c1db1f-b701-45bd-96e0-a313ee3430b3",
              "tenant_id": "",
              "extra_dhcp_opts": [],
              "device_owner": "network:router_gateway",
              "mac_address": "fa:16:3e:58:42:ed",
              "fixed_ips": [
                  {
                      "subnet_id": "008ba151-0b8c-4a67-98b5-0d2b87666062",
                      "ip_address": "172.24.4.2"
                  }
              ],
              "id": "d80b1a3b-4fc1-49f3-952e-1e2ab7081d8b",
              "security_groups": [],
              "device_id": "9ae135f4-b6e0-4dad-9e91-3c223e385824"
              "created_at": 2015-07-31T11:50:05
              "updated_at": 2015-07-31T11:51:12
          }
      ]
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

Take port as an example, the command looks like:

::

  neutron port-list --changed-since 2015-07-31T00:00:00

Performance Impact
------------------

Performance of syncing resource status with other systems like a monitor system
or a user interface can be improved. Instead of retrieving all the resources in
every syncing period to update resource status, these systems can just retrieve
resources updated during the syncing interval using change_since filter.

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
projects in OpenStack like `Nova`_, `Cinder`_ have timestamp fields to track
the operation time of resources.

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
  caizhiyuan1@huawei.com

Other contributors:
  TBD

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
* Test if change_since filter can be correctly applied.

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
.. _`Cinder`: https://github.com/openstack/cinder/blob/master/cinder/db/sqlalchemy/models.py#L35

