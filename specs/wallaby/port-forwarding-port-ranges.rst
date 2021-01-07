..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Floating IP port forwarding to support port ranges
==================================================

https://blueprints.launchpad.net/neutron/+spec/floatingips-portforwarding-ranges

The use of port ranges in NAT eases the rules creation process as it replaces
the use of many entries in the iptables by just one. Also, some users, when
requesting virtual machines that have some ports exposed to the Internet,
prefer to request a range of ports to be exposed to avoid doing many requests
for operators regarding the networking configurations. Therefore, they usually
request a slice of external ports to deploy their applications, so they will
be able to choose and change the ports they desire on the fly without needing
to contact anyone else or configuring it in OpenStack.

Moreover, there are applications such as FTP, video conference systems,
streaming, and others that use more than one port (ports in sequence).

Therefore, it is easier to enable all of them (the requested/needed port range)
in a single configuration.

Problem Description
===================

* Currently, if a user wants to create NAT rules that cover multiple ports,
  he/she needs to create them one by one, which is cumbersome in some use
  cases.
* When configuring NAT via iptables [1]_ (and other networking
  systems/applications), operators can use port ranges while configuring a
  rule; therefore, it is natural for network engineers to handle/use port
  ranges in their NAT configurations.

Proposed Change
===============

Change the Floating IP port forwarding API to allow the use of port ranges to
create NAT rules. The changes are presented as follows. Today, when a user is
creating a port forwarding rule, he/she creates a rule via API, or CLI and
sends a JSON like:

.. code-block:: json

    {
      "port_forwarding": {
        "protocol": "tcp",
        "internal_ip_address": "172.16.0.7",
        "internal_port": 80,
        "internal_port_id": "b67a7746-dc69-45b4-9b84-bb229fe198a0",
        "external_port": 8080,
        "description": "desc"
      }
    }

The external/internal ports are integers and if the user desires to create a
port forwarding in a floating ip for many ports, he/she will need to resend
this JSON many times to fit the number of ports he/she desires.

The proposal is to change the API/CLI to accept a string in external/internal
ports range. This will allow users to create a rule with port ranges (instead
of executing multiple calls). Therefore, a user would not need anymore to
create 100 rules to reach 100 port forwards in his/her virtual machine, he/she
could just send a JSON like that:

.. code-block:: json

    {
      "port_forwarding": {
        "protocol": "tcp",
        "internal_ip_address": "172.16.0.7",
        "internal_port_range": "8000:8100",
        "internal_port_id": "b67a7746-dc69-45b4-9b84-bb229fe198a0",
        "external_port_range": "9000:9100",
        "description": "desc"
      }
    }

The API will be still accepting the ``internal_port`` and ``external_port``
attributes to maintain backward compatibility.

The new expected response will be a JSON like:

.. code-block:: json

    {
      "port_forwarding": {
        "protocol": "tcp",
        "internal_ip_address": "172.16.0.7",
        "internal_port_range": "8000:8100",
        "internal_port_id": "b67a7746-dc69-45b4-9b84-bb229fe198a0",
        "external_port_range": "9000:9100",
        "description": "desc",
        "id": "825ade3c-9760-4880-8080-8fc2dbab9acc"
      }
    }

New validations will be created to avoid users to enter invalid ranges in
ports. Here follows a summary of all of the validations executed by the system:

* Only N(external port[s]) to N (internal port[s]) are allowed or
  N(external port[s]) to 1 (internal port). Therefore, all of the other
  combinations (1:N, N:M, M:N) are not allowed and will generate an error.
* When defining a port(s) range, the ports in this range cannot be already
  in use by any other port forwarding rule for the same floating IP and
  protocol.
* A valid port is a number between 1 and 65535. All other cases will generate
  an error. Even though "0" is a valid port, we will not allow its use as it is
  reserved.
* The notation to define a port range is the following:
  <port_range_begin>[:<port_range_end>]; anything that does not match this
  definition will generate an error.
* When doing the request, the user can choose between sending a JSON with
  ``internal_port_range`` (string) or ``internal_port`` (integer), the same to
  the external ports, sending both ``internal_port_range`` and
  ``internal_port`` will raise a validation error.

Data Model Impact
-----------------

Today, the port_forwarding table schema is something like:

+----+---------------+---------------+--------------------------+----------+-----------------+
| id | floatingip_id | external_port | internal_neutron_port_id | protocol |      socket     |
+====+===============+===============+==========================+==========+=================+
| A1 |      C2       |      80       |            A2            |    tcp   | 172.16.0.7:8080 |
+----+---------------+---------------+--------------------------+----------+-----------------+
| B1 |      C2       |      81       |            B2            |    tcp   | 172.16.0.7:8081 |
+----+---------------+---------------+--------------------------+----------+-----------------+

To make the port_forwarding table more like the PortForwarding object and the
JSON, the port_forwarding table will be updated splitting the socket column
into three new columns (internal_ip_address, internal_port_start,
internal_port_end), also, the external_port column will be split into two new
columns (external_port_start and external_port_end). The new table would be like below:

+----+---------------+---------------+---------------------+-------------------+--------------------------+----------+---------------------+---------------------+-------------------+-----------------+
| id | floatingip_id | external_port | external_port_start | external_port_end | internal_neutron_port_id | protocol | internal_ip_address | internal_port_start | internal_port_end |      socket     |
+====+===============+===============+=====================+===================+==========================+==========+=====================+=====================+===================+=================+
| A1 |      C2       |      80       |         80          |        80         |            A2            |    tcp   |     172.16.0.7      |         8080        |        8080       | 172.16.0.7:8080 |
+----+---------------+---------------+---------------------+-------------------+--------------------------+----------+---------------------+---------------------+-------------------+-----------------+
| B1 |      C2       |      81       |         81          |        81         |            B2            |    tcp   |     172.16.0.7      |         8081        |        8081       | 172.16.0.7:8081 |
+----+---------------+---------------+---------------------+-------------------+--------------------------+----------+---------------------+---------------------+-------------------+-----------------+

The legacy data migration would be done with an alembic migration upgrade
script.

We have 5 steps in the data migration:

1) Create the ``internal_ip_address``, ``internal_port_start``, ``external_port_start``,
   ``internal_port_end``, ``external_port_end`` columns with NULL values;

2) Select id, socket and external_port columns for every table entry;

3) For each entry, split the socket value by ':' and put the result in the new
   ``internal_ip_address`` and ``internal_port_start`` columns from same id as the
   socket. Migrate the value of the ``internal_port_start`` column in the
   ``internal_port_end`` and migrate the value of the ``external_port`` column
   in the ``external_port_start`` and ``external_port_end`` columns;

4) Change the socket column to nullable;

5) Change the external_port column to nullable;

Sub Resource Extension
----------------------

It will be created as an extension that overrides the parameters (external_port
and internal_port) to also accept port ranges in their validations. The
attributes map of new sub resource would be like:

.. code-block:: python

    SUB_RESOURCE_ATTRIBUTE_MAP = {
        'port_forwarding': {
            'parameters': {
                'external_port_range': {
                    'allow_post': True, 'allow_put': True,
                    'validate': {'type:port_range': None},
                    'is_visible': True,
                    'is_sort_key': True,
                    'is_filter': True},
                'internal_port_range': {
                    'allow_post': True, 'allow_put': True,
                    'validate': {'type:port_range': None},
                    'is_visible': True},
            }
        }
    }

REST API Impact
---------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignees:

* Pedro <phpm13@gmail.com>

* Rafael <rafael@apache.org>

Other contributors:

Work Items
----------
1) API extension (neutron-lib)
2) DB extension/data migration (neutron)
3) Extend Neutron API with new validations for port ranges (neutron)
4) Extend l3 agent to apply the ranges in the iptable rule (neutron)
5) Extend the ``floating ip port forwarding`` command to accept and validate ranges in the CLI (python-openstackclient)
6) Change the port types from int to string in the PortForwarding object (openstacksdk)
7) Tests
8) Documentation

To be done in future releases
-----------------------------
1) Remove the columns "socket" and "external_port" from the port_forwarding
table in the neutron's database, as these columns are no more used after the
release wallaby.

Dependencies
============

None


References
==========

.. [1] https://www.netfilter.org/documentation/HOWTO/NAT-HOWTO-6.html