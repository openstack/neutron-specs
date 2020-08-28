..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================
Source and destination filtering for metering label rules
=========================================================
This spec adds source and destination filtering options
to Neutron metering label rules.

Problem Description
===================
Neutron metering label rules have a parameter called "remote-ip-prefix", which
would allow operators to filter traffic based on the remote IP address.
However, since [1]_, its meaning was changed to the exact opposite, which makes
a bit of confusion. Instead of matching on the remote prefix (towards the
external interface), it matches the local prefix (towards the OpenStack tenant
networks).

Ideally, to satisfy the use case presented in [1]_ (which was achieved by
inverting the use of "remote-ip-prefix"), operators should be able to create
rules based on local-ip-prefix and remote-ip-prefix.

Proposed Change
===============
As discussed in the Neutron drivers metering that approved the RFE [2]_ of
this spec, we  will deprecate the parameter `remote-ip-prefix` of Neutron
metering API.

Therefore, we will be introducing two new parameters with this spec in the
Neutron metering rule API. These new parameters will be called
"source_ip_prefix", and "destination_ip_prefix" (like in IPtables).
The behavior of "remote_ip_prefix" will be maintained, but we will fix its
documentation and mark it for removal in future releases [1]_.

The "source_ip_prefix" and "destination_ip_prefix" could be used together, or
only one of them can be defined. However, a metering rule must always have at
least one of them (source_ip_prefix or destination_ip_prefix) defined. On the
other hand, these two new parameters will not be allowed to be used in
conjunction with "remote_ip_prefix".

API JSON
--------
Current JSON  for "v2.0/metering/metering-label-rules" endpoint:

.. code-block:: json

    {
      "remote_ip_prefix": "192.168.0.14/32",
      "direction": "egress",
      "metering_label_id": "9ffd6512-9d2a-4dd2-9657-6a605126264d",
      "id": "f1694467-d866-4d8e-a8dc-18da516caedc",
      "excluded": false
    }

Adding new attributes:

.. code-block:: json

    {
      "source_ip_prefix": "192.168.0.14/32",
      "destination_ip_prefix": "0.0.0.0/0",
      "direction": "egress",
      "metering_label_id": "9ffd6512-9d2a-4dd2-9657-6a605126264d",
      "id": "f1694467-d866-4d8e-a8dc-18da516caedc",
      "excluded": false
    }

Database table changes
----------------------

Currently, the table "meteringlabelrules" is defined as:

.. code-block:: bash

    +-------------------+--------------------------+------+-----+---------+-------+
    | Field             | Type                     | Null | Key | Default | Extra |
    +-------------------+--------------------------+------+-----+---------+-------+
    | id                | varchar(36)              | NO   | PRI | NULL    |       |
    | direction         | enum('ingress','egress') | YES  |     | NULL    |       |
    | remote_ip_prefix  | varchar(64)              | YES  |     | NULL    |       |
    | metering_label_id | varchar(36)              | NO   | MUL | NULL    |       |
    | excluded          | tinyint(1)               | YES  |     | 0       |       |
    +-------------------+--------------------------+------+-----+---------+-------+

We would add two new fields to it. Therefore, it would look like:

.. code-block:: bash

    +-----------------------+--------------------------+------+-----+---------+-------+
    | Field                 | Type                     | Null | Key | Default | Extra |
    +-----------------------+--------------------------+------+-----+---------+-------+
    | id                    | varchar(36)              | NO   | PRI | NULL    |       |
    | direction             | enum('ingress','egress') | YES  |     | NULL    |       |
    | remote_ip_prefix      | varchar(64)              | YES  |     | NULL    |       |
    | source_ip_prefix      | varchar(64)              | YES  |     | NULL    |       |
    | destination_ip_prefix | varchar(64)              | YES  |     | NULL    |       |
    | metering_label_id     | varchar(36)              | NO   | MUL | NULL    |       |
    | excluded              | tinyint(1)               | YES  |     | 0       |       |
    +-----------------------+--------------------------+------+-----+---------+-------+

Neutron Metering agent changes
------------------------------

The IPtables driver in the metering agent will need to handle the new
parameters "source_ip_prefix" and "destination_ip_prefix" properly.
When building the IPtable rules the parameter "destination_ip_prefix"
(if defined) will be used with the option "-d" (IPtables option).
On the other hand, the parameter "source_ip_prefix" (if defined) will
be used with option "-s"(IPtables option).

Validations
-----------
To simplify validations, we propose to remove the overlapping IP_prefixes
validations for the new fields (for the remote IP prefix, it will be
maintained). The rationally behind this removal is that if the operator
wants to somehow create rules that overlap, we should not be the ones blocking
it (there might be some business logic that needs it).

We will implement the following validation:
* The source IP prefix must be a valid IPv4 CIDR
* The destination IP prefix must be a valid IPv4 CIDR
* Each metering label rule requires at least source, destination, or remote IP
prefix to be informed. The remote IP prefix is marked as deprecated.
Therefore, once it is removed, the users will have to enter at least source or
destination IP prefixes. One can also use both (source and destination IP
prefixes) to build rule.
* source and destination IP prefixes cannot be used in conjunction with
remote IP prefix.

API impacts
-----------
Two new parameters will be introduced, but they are not required.
Therefore, people using it would not suffer an immediate impact.
However, when the "remote_ip_prefix" is removed, people might
have a problem. therefore, as soon as the new method of building rules
is available, people will be encouraged to use it, instead of the
"remote_ip_prefix" metering rule base.


Assignee(s)
-----------

Primary assignees:
 - Rafael <rafael@apache.org>

Other contributors:

Work Items
----------

The following are the work items for the planned release.

1) Deprecate remote IP prefix (Neutron-lib)

 - Deprecate remote IP prefix

 - Fix documentation

2) Add source and destination attributes (Neutron-lib) -- executed via [3]_

 - Add new attributes in api/definitions/metering.py

 - Fix JSON of examples and documentation

3) Deprecate remote IP prefix (Neutron)

 - Deprecate remote IP prefix

 - Fix documentation

 - Log a warning when people use it

4) Change execution flow in Neutron and Neutron metering agent to use the new fields. (Neutron)

 - Add the new DB fields in objects.metering.MeteringLabelRule and neutron/db/models/metering.MeteringLabelRule

 - DB script in neutron/db/migration/alembic_migrations/versions/victoria

 - Actual implementation to use the new attributes, and unit tests

 - Update the documentation of the API with the new fields

5) Deprecate remote IP prefix (OpenStack SDK)

 - Deprecate remote IP prefix

 - Fix documentation

 - Log a warning when people use it


6) Deprecate remote IP prefix (OpenStack python client)

 - Deprecate remote IP prefix

 - Fix documentation

 - Log a warning when people use it

7) Add the new fields (OpenStack SDK)

 - Add the new fields

 - Fix documentation

8) Add the new fields (OpenStack python client)

 - Add the new fields

 - Fix documentation

After we finish all of these items, in a future release, we will need to
execute the following removal items.

1) Remove remote IP prefix (Neutron-lib)

 - Fix JSON of examples and documentation

2) Remove remote IP prefix (Neutron)

 - Fix documentation

3) Remove remote IP prefix (OpenStack SDK)

 - Fix documentation

4) Remove remote IP prefix (OpenStack python client)

 - Fix documentation

Dependencies
============

None


References
==========

.. [1] https://opendev.org/openstack/neutron/commit/92db1d4a2c49b1f675b6a9552a8cc5a417973b64
.. [2] https://bugs.launchpad.net/neutron/+bug/1889431
.. [3] https://review.opendev.org/#/c/743828/
