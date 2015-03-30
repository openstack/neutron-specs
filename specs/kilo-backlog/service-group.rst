..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


========================================
Service group and Service Object support
========================================

https://blueprints.launchpad.net/neutron/+spec/fwaas-customized-service

In a traditional networking system, the term "service" refers to a type of connection
between two endpoints that is visible as it passes through the networking system.
The "service" is most commonly described as a layer 4 protocol number combined with
source and destination ports, all of which are visible from the L4 payload.
An extension is being proposed in this blueprint that allows the network administrators
to create service objects.
The customized service objects can then be grouped together to form a service group.
The service object and service group abstractions proposed in this spec will be first consumed
by the FWaaS implementation, but its applicability is not limited to FWaaS and can be leveraged
in other features where such service definition and grouping is required.

Problem Description
=======================
1. In FWaaS, an administrator can use port range and protocol in a firewall rule
to define the traffic type. But we don't have a flexible way to allow user to specify more
than one type of traffic in the same rule. To support different traffic type, with the
same source, destination address and action, different rules need to be created.
This makes the process of defining firewall rules un-scalable.
2. Many vendors' (eg. Palo Alto Networks, Juniper) implement security policy using service definitions
and do not configure protocol and port on the policy directly. We should support
the same in the firewall rule for easy integration. As such, adding support for this extension
will provide a consistent abstraction for vendors that want to support configuring firewall rules
via service definitions.
3. Some vendors also support special attributes for different traffic type, one usage is to
allow session idle timeout values for different traffic type. We cannot support this today.

Proposed Change
==================

We propose to adding a new extension with two resources:
. Service Groups
. Service Objects

A user can define type of traffic using a Service Object.Service Objects can be grouped
into Service Groups and referenced from a FWaaS Firewall Rule.
Each service object can be optionally defined with a timeout value that can be used to overwrite
default session idle timeout value.

When being used with firewall rules, the service groups are allowed to be reused among
different firewall rules in the same tenant.

To simplify the initial implementation, following features are not supported in the first
implementation:
1. support of sharing the service object among service groups.
2. support of the case where the cloud provider administrator creates service
group for tenant to use.
We should be able to add these two features later with backward compatible APIs.

The service group can only be deleted if there are no other objects referencing it and
in the current case, the referencing object will be firewall rules.

A service object can only be deleted if there are no other service groups referencing
it.

Since most of the firewall vendors support only using service group or service object to
configure firewall policy and do not allow a user to configure the protocol and port on the
security policy directly, we might deprecate the protocol and port options from the
firewall rule resources. But we will delay this decision until later releases.

Even though the current document is targeting firewall rule as the user of the service group, the
service group could also be useful when configure security group or group policy. In
the developer session of Hong Kong openstack summit, some of developers suggested to
make service group as global resource. Based on this suggestion, we will make the
service group and service object global resources.


Here is an example for using service group in firewall rule:

Assuming a tenant has two servers that provide certain type of web services on port 8080,
80 and 8000. We can create two firewall rules to permit traffic from any source ip address
to these two servers. The service provided from 8000 port has very short idle timeout
(10 seconds), the services provided on port 8080 and 80 have default idle timeout

neutron service-group-create http-services
This will create a service group named as http-services

neutron service-object-create --protocol tcp --destination-port 8080 http_object_8080 \
--service-group http-services
This creates a service object named http_object_8080 in the http-service group

neutron service-object-create --protocol tcp --destination-port 80   http_object_80 \
--service-group http-services
This creates a service object named http_object_80 in http-service group

neutron service-object-create --protocol tcp --destination-port 8000 --timeout 10 \
http_object_8000 --service-group http-services

This creates a service object named http_object_8000 in http-service group, The
service idle timeout for this object is 10 seconds. It implies the firewall session
that created by this type of traffic has an idle timeout as 10 seconds (comparing the
default timeout 1800 seconds)

neutron firewall-rule-create --destination-ip-address 10.0.2.1 --service-group \
http-services action permit
neutron firewall-rule-create --destination-ip-address 11.0.2.1 --service-group \
http-services action permit

These two rules permit traffic from any IP address to service 10.0.2.1 and 11.0.2.1
that match any service defined in the service group http-services

In the current reference design, when the firewall rules get pushed to the firewall
agent, the plugin already expands firewall rules along with service group content.


Note:
1. firewall rule can also be configured with protocol and port range, for current
reference design, we will not allow service-group, protocol and port range to be configured
together.


Data Model Impact
-----------------

Firewall rules:

+-------------------+------------+-----------------+-----------+------+-------------------------+
| Attribute name    |  Type      | Default Value   | Required  | CRUD | Description             |
+-------------------+------------+-----------------+-----------+------+-------------------------+
| service_groups    | List       |  empty          | N         | CRU  | List of service groups  |
+-------------------+------------+-----------------+-----------+------+-------------------------+

Service group:

+-------------------+------------+-----------------+-----------+------+-------------------------+
| Attribute name    |  Type      | Default Value   | Required  | CRUD | Description             |
+-------------------+------------+-----------------+-----------+------+-------------------------+
| id                | uuid       |  generated      | Y         |  R   |                         |
+-------------------+------------+-----------------+-----------+------+-------------------------+
| name              | String     |  empty          | N         | CRU  |Name of service group    |
+-------------------+------------+-----------------+-----------+------+-------------------------+
| description       | String     |  empty          | N         | CRU  |                         |
+-------------------+------------+-----------------+-----------+------+-------------------------+
| tenant id         | uuid       |  empty          | Y         | R    |Id of tenant that creates|
|                   |            |                 |           |      |service group            |
+-------------------+------------+-----------------+-----------+------+-------------------------+
| service objects   | list       |  empty list     | N         | CRU  |List of service objects  |
+-------------------+------------+-----------------+-----------+------+-------------------------+

Service object:

+----------------------+----------------+-----------------------------+------+--------------------------+
| Attribute name       | Type           | Default Value   | Required  | CRUD |Description               |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| id                   | uuid           |  generated      | Y         | R    |                          |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| name                 | String         |  empty          | N         | CRU  |Service object name       |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| service group id     | uuid           |  empty          | N         | CRU  |Foreign key to service grp|
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| protocol             | string         |  empty          | Y         | CRU  |'tcp','udp','icmp','any'  |
|                      |                |                 |           |      | or protocol id (0-255)   |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| source_port          | integer or str |  empty          | N         | CRU  |This could be either a    |
|                      |                |                 |           |      |single port (integer or   |
|                      |                |                 |           |      |string) or a range(string)|
|                      |                |                 |           |      |in the form "p1:p2"       |
|                      |                |                 |           |      |where(0<=p1<=p2 <=65535)  |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| destination_port     |integer or str  |  empty          | N         | CRU  | Same as source_port      |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| icmp_code            | char           |  empty          | N         | CRU  |                          |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| icmp_type            | char           |  empty          | N         | CRU  |                          |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| timeout              | short          |  empty          | N         | CRU  |                          |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| tenant_id            | uuid           |  empty          | Y         | R    |                          |
+----------------------+----------------+-----------------+-----------+------+--------------------------+


New CLIs:
service-group-create

service-group-delete

service-group-list

service-group-show

service-group-update

service-object-create

service-object-delete

service-object-list

service-object-show


REST API Impact
---------------
The new resources:

.. code-block:: python

  RESOURCE_ATTRIBUTE_MAP = {
      'service_groups': {
          'id': {'allow_post': False, 'allow_put': False,
                 'validate': {'type:uuid': None},
                 'is_visible': True,
                 'primary_key': True},
          'name': {'allow_post': True, 'allow_put': True,
                   'is_visible': True, 'default': '',
                   'validate': {'type:name_not_default': None}},
          'description': {'allow_post': True, 'allow_put': True,
                          'is_visible': True, 'default': ''},
          'tenant_id': {'allow_post': True, 'allow_put': False,
                        'required_by_policy': True,
                        'is_visible': True},
          'service_objects': {'allow_post': False, 'allow_put': False,
                              'convert_to': attr.convert_none_to_empty_list,
                              'is_visible': True},

      }

      'service_objects': {
          'id': {'allow_post': False, 'allow_put': False,
                 'validate': {'type:uuid': None},
                 'is_visible': True, 'primary_key': True},
          'name': {'allow_post': True, 'allow_put': True,
                   'is_visible': True, 'default': '',
                   'validate': {'type:name_not_default': None}},
          'service_group_id': {'allow_post': True, 'allow_put': False,
                               'is_visible': True, 'required_by_policy': True},
          'protocol': {'allow_post': True, 'allow_put': False,
                       'is_visible': True, 'default': None,
                       'convert_to': _convert_protocol},
          'source_port': {'allow_post': True, 'allow_put': False,
                          'validate': {'type:service_port_range': None},
                          'convert_to': _convert_port_to_string,
                          'default': None, 'is_visible': True},
          'destination_port': {'allow_post': True, 'allow_put': False,
                               'validate': {'type:service_port_range': None},
                               'convert_to': _convert_port_to_string,
                               'default': None, 'is_visible': True},
          'icmp_code': {'allow_post': True, 'allow_put': False,
                        'validate': {'type:icmp_code': None},
                        'convert_to': _convert_icmp_code,
                        'default': None, 'is_visible': True},
          'icmp_type': {'allow_post': True, 'allow_put': False,
                        'validate': {'type:icmp_type': None},
                        'convert_to': _convert_icmp_type,
                        'default': None, 'is_visible': True},
          'timeout': {'allow_post': True, 'allow_put': False,
                      'validate': {'type:range': [0, 65535]},
                      'convert_to': attr.convert_to_int,
                      'default': 0, 'is_visible': True},
          'tenant_id': {'allow_post': True, 'allow_put': False,
                        'required_by_policy': True,
                        'is_visible': True},

      }

  }

  RESOURCE_ATTRIBUTE_MAP = {
    'firewall_rules': {
        'service_groups': {'allow_post': True, 'allow_put': True,
                           'convert_to': attr.convert_none_to_empty_list,
                           'default': None, 'is_visible': True},
    }
  }

+---------------+----------------------------+----------------------+
|Object         |URI                         |Type                  |
+---------------+----------------------------+----------------------+
|service group  |/service-groups             |GET                   |
+---------------+----------------------------+----------------------+
|service group  |/service-groups             |POST                  |
+---------------+----------------------------+----------------------+
|service group  |/service-groups/{id}        |GET                   |
+---------------+----------------------------+----------------------+
|service group  |/service-groups/{id}        |PUT                   |
+---------------+----------------------------+----------------------+
|service group  |/service-group/s{id}        |DELETE                |
+---------------+----------------------------+----------------------+
|service object |/service-objects            |GET                   |
+---------------+----------------------------+----------------------+
|service object |/service-objects            |POST                  |
+---------------+----------------------------+----------------------+
|service object |/service-objects/{id}       |GET                   |
+---------------+----------------------------+----------------------+
|service object |/service-objects/{id}       |PUT                   |
+---------------+----------------------------+----------------------+
|service object |/service-objects/{id}       |DELETE                |
+---------------+----------------------------+----------------------+

Security Impact
---------------
* Does this change touch sensitive data such as tokens, keys, or user data?
  No

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?
  No

* Does this change involve cryptography or hashing?
  No

* Does this change require the use of sudo or any elevated privileges?
  No

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.
  Yes

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.
  No

Notifications Impact
--------------------
None

Other End User Impact
---------------------
None

Performance Impact
------------------
None

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
This model and framework could be utilized by various modules for example VPNaaS, LBaaS,
security group as it provides the correct data modeling and reduces code bloat.

Alternatives
------------
Without service group, administrator can create separate rule for each type of traffic.

The issue with this method is high overheads, it may create way too many rules with the
duplicated resource defined in it.

Also, the most of firewall vendors have service group like concept in their policy
definition. Adding the notion of the service group in the firewall rule simplifies
integration path for firewall vendors

Implementation
==============

Assignee(s)
-----------
Primary assignee:
  badveli_vishnuus@yahoo.com
  beyounn@gmail.com

Work Items
------------
* API and database
* Reference implementation will be based on existing fwaas reference implementation using iptables.
* python-neutronclient


Dependencies
============
None

Testing
=======

Both Tempest and Functional tests will be used.

Tempest Tests
-------------
Tempest spec:
https://review.openstack.org/#/c/134062/
Complete API coverage is included.
https://review.openstack.org/#/c/113409/
Complete unit test coverage of the code is included.
https://review.openstack.org/#/c/106274/

Functional Tests
----------------
Complete test coverage of the code is included.
https://review.openstack.org/#/c/106274/

API Tests
---------
Not applicable.

Documentation Impact
====================

User Documentation
------------------
Documentation for both administrators and end users will have to be
updated. Administrators will need to know how to configure the service
group by using the service group API and python-neutronclient.

Developer Documentation
-----------------------
None needed beyond documentation changes listed above.

References
==========
None


