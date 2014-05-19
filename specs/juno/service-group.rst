..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


========================================
Service group and Service object support
========================================
Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/fwaas-customized-service

In the traditional firewall design a service is used to define type of traffic
in firewall. This blueprint creates an extension that allows the firewall
administrators to create customized service objects. The customized service
objects can be grouped together to form a service group object.

Problem description
=======================
1. In FWaaS, administrator can use port range and protocol inside firewall rules
to define traffic type. But we don't have a flexible way to allow user to specify more
than one type of traffic in the same rule. To support different traffic type, with the
same source, destination address and action, different rules need to be created.
This makes the process of defining firewall rules un-scalable.
2. Most of vendors' (eg. PAN, Juniper) security policy is implemented based on
service and do not configure protocol and port on the policy directly. We should support
the same in the firewall rule for the easy integration.
3. Some vendors also support special attributes for different traffic type. One of the
usage is to allow session idle timeout values for different traffic type. We can not support
this today.

Proposed change
==================

We propose to add a new extension with two resources. One is called service group
and one is called service object.

Administrator can use service object to define a special type of traffic. The service
objects are grouped into a service group that can be used across neutron features, with
the initial usage being in firewall rules exposed by the FWaaS feature.

Each service object can be defined with a timeout value that can be used to override
default session idle timeout value. In the FWaaS reference implementation, this timeout
will be implemented by using the timeout option in the iptable connection tracking.

When being used with firewall rules, the service groups are allowed to be reused among
different firewall rules in the same tenant.

To simplify the initial implementation, following features are not supported in the first
implementation:
1. support of sharing the service object among service groups.
2. support of the case where the cloud provider administrator creates service
group for tenant to use.
We should be able to add these two features later with backward compatible APIs.

The service group can only be deleted if there are no other objects referencing it and
in the current case, the referencing object will be firewall rules. A service object can
only be deleted if there are no other service groups are referencing it.

Since most of the firewall vendors support only using service group or service object to
configure firewall policy and do not allow user to configure the protocol and port on the
security policy directly, we can potentially deprecate the protocol and port options from
the firewall rule resources. But we will delay this decision until later releases.

Even though the current document is targeting firewall rule as the user of the service
group, the service group could also be useful when configure security group or group policy.
In the developer session of Hong Kong openstack summit, some of developers suggested to
make service group as global resource. Based on this suggestion, we will make the
service group and service object an independent extension module, such that its usage
is not limited to FWaaS.

Naming:
The names of service group and service object are selected based on the fact that many
firewall vendors are using the same name for the same feature in their firewall products.
Also, traditionally, in UNIX system, all supported protocol and port are listed in
/etc/services file.

Use case:
An administrator wants to create firewall rules to allow all H.323 traffics to server 2.2.2.2.
The H.323 traffic can be on following port/protocol pairs:

tcp/1720
tcp/1503
tcp/389
tcp/522
tcp/1731
udp/1719

Without service group, administrator has to create 6 different rules and other than
protocol and port, all other fields in each rule will be a duplication to the same fields in
other rules. Basically, administrator needs to create separated rules for each type of the
traffics that can hit on the server 2.2.2.2 and many of the rules are duplicated. With service
group, administrator only needs to create a few service groups and there will not be any
duplications among firewall rules. For the current use case, administrator only needs to create
one service group and one firewall rule. This can reduce amount of firewall rules.

Here is an example for using service group in firewall rule:
Assuming a tenant has two servers that provide certain type of web services on port 8080,
80 and 8000. We can create two firewall rules to permit traffic from any source IP address
to these two servers. The service provided from 8000 port has very short idle timeout
(10 seconds), the services provided on port 8080 and 80 have default idle timeout.

neutron service-group-create tcp-http-services
This will create a service group named as tcp-http-services

neutron service-object-create --protcol tcp --destination-port 8080 http_object_8080 \
--service-group tcp-http-services
This creates a service object named http_object_8080 in the http-service group

neutron service-object-create --protcol tcp --destination-port 80   http_object_80   \
--service-group tcp-http-services
This creates a service object named http_object_80 in tcp-http-services group

neutron service-object-create --protcol tcp --destination-port 8000 --timeout 10 \
http_object_8000 --service-group tcp-http-services
This creates a service object names http_object_8000 in tcp-http-services group, the
service idle timeout for this object is 10 seconds. It implies the firewall session
that created by this type of traffic has idle timeout as 10 seconds (comparing the
default timeout 1800 seconds).

neutron firewall-rule-create --destination-ip-address 10.0.2.1 --service-group \
tcp-http-service --action permit
neutron firewall-rule-create --destination-ip-address 11.0.2.1 --service-group \
tcp-http-service --action permit
These two rules permit traffic from any IP address to service 10.0.2.1 and 11.0.2.1
that match any service defined in the service group tcp-http-services.

In the current reference design, when the firewall rules get pushed to the firewall
agent, the agent checks if the rule is referencing service groups (by the service
group id), if it is, then the agent queries the service group content from plugin
and expand the firewal rule based on the content of the service group into the
iptables. Since the firewall policy is pushed with all the rules together, it would
be better to have agent to query the service group contents so that the policy push
message will not be too big. When deleting rules, the agent will do the same so that
it can recover the original iptable rules and delete them.


Note:
1. firewall rule can also be configured with protocol and port range, for current
reference design, we will not allow service-group, protocol and port range to be configured
together.
2. Later, we can used IPset to implement firewall reference design, in that way, it will
be much easier for us to apply service group.


Alternatives
------------
Without service group, administrator can create separate rule for each type of traffic.

The issue with this method is high overheads, it may create way too many rules with the
duplicated resource defined in it.

Also, the most of firewall vendors have service group like concept in their policy
definition. Adding the notion of the service group in the firewall rule simplifies
integration path for firewall vendors

Data model impact
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
| service objects   | list       |  empty list     | N         | R    |List of service objects  |
+-------------------+------------+-----------------+-----------+------+-------------------------+

Service object:

+----------------------+----------------+-----------------------------+------+--------------------------+
| Attribute name       | Type           | Default Value   | Required  | CRUD |Description               |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| id                   | uuid           |  generated      | Y         | R    |                          |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| name                 | String         |  empty          | N         | CRU  |Service object name       |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| protocol             | string         |  empty          | Y         | CR   |'tcp','udp','icmp','any'  |
|                      |                |                 |           |      | or protocol id (0-255)   |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| source_port          | integer or str |  empty          | N         | CR   |This could be either a    |
|                      |                |                 |           |      |single port (integer or   |
|                      |                |                 |           |      |string) or a range(string)|
|                      |                |                 |           |      |in the form "p1:p2"       |
|                      |                |                 |           |      |where(0<=p1<=p2 <=65535)  |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| destination_port     |integer or str  |  empty          | N         | CR   | Same as source_port      |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| icmp_code            | char           |  empty          | N         | CR   |  ICMP code number        |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| icmp_type            | char           |  empty          | N         | CR   |  ICMP type number        |
+----------------------+----------------+-----------------+-----------+------+--------------------------+
| timeout              | short          |  empty          | N         | CR   | idle timeout in seconds  |
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

service-object-update

REST API impact
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
|service group  |/service-groups/{id}        |DELETE                |
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

Security impact
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

Notifications impact
--------------------
None

Other end user impact
---------------------
None

Performance Impact
------------------
None

Other deployer impact
---------------------
None

Developer impact
----------------
None


Implementation
==============

Assignee(s)
-----------
Yi Sun beyounn@gmail.com
Vishnu Badveli breddy@varmour.com

Work Items
------------
* API and database
* Reference implementation
* python-neutronclient


Dependencies
============
None

Testing
=======
Both unit test and tempest test will be required


Documentation Impact
====================
Documentation for both administrators and end users will have to be
provided. Administrators will need to know how to configure the service
group by using the service group API and python-neutronclient.



References
==========
None


