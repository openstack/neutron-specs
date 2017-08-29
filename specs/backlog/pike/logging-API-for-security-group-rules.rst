..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Logging API for security-group-rules
====================================

https://bugs.launchpad.net/neutron/+bug/1468366

This spec implements a way to capture and store events related to security
groups.

Problem Description
===================

*Operators* (including cloud admin and developers) or projects want to store
logs of network traffic of:

* North-south network traffic travels between an instance and external network.
* East-west network traffic travels between instances.

To detect illegal communication or attack patterns, alert abnormal activities
for compliance reason.

In addition, these logs can also be used for debugging security groups (help
project make sure security group rules work as expected or not). However
this is not the main goal since other approaches present (e.g TaaS, tcpdump)
to fix the same problem.

In Neutron, all traffic allowed/dropped on instances are managed by security
groups. However, logging is currently a missing feature in security groups.

Proposed Change
===============

To address this problem, we'd like to propose a logging API with objective:

* **Define format and ways to collect events related to security groups.**

Logging API will collect ACCEPT or DROP or both events related to security
groups configured on port(s). Please refer to `Expected API behavior`_ for
more details.

This spec implements logging API for security groups for OVS native firewall
(OVS) and Iptables-based (Linux Brigde) for *operator-only*. It also lays out
a model which can be extended to other resources (e.g Firewall logging) easily.

Initially, we have both PoC for OVS native firewall and Iptables-based logging.
The OVS native firewall logging will be completely supported first as it  is
now becoming the default driver, Iptables-based logging will be finished later.
This API is considered to work consistently between OVS native firewall and
Iptables-based logging.

Regarding **multiple driver environment**, this feature will have a similar
validation like QoS [1]_ as a first approach. [2]_

Logging implementation
----------------------

Logging API is designed as a service plugin. It is defined as a generic logging
API for resources such as security groups and firewall.

In server side, the class design for LoggingApiPlugin would look like::

                       +-----------------------+
                       | LoggingApiPluginBase  |
                       +----------^------------+
                                  |
                       +-----------------------+
                       |  LoggingApiPlugin     |
                       +-----------------------+
                       |  _init_()             |
                       |  get_logs()           |
                       |  get_log()            |
                       |  create_log()         |
                       |  update_log()         |
                       |  delete_log()         |
                       +-----------------------+

The reference implementation can be found in [3]_.

The LoggingExtension is an agent extension. It is common for all logging
resources like security groups and firewall. The LoggingExtension will receive
CREATED/UPDATED/DELETED events of the logging resource and pass these events
to logging drivers. Each logging driver defines the resources it supports.
In case of security group, we can have a security group logging driver implemented
as OVSFirewallLogging or IptablesLoggingDriver.

Class design of LoggingExtension would look like::

                      +-----------------------+
                      |     AgentExtension    |
                      +------------^----------+
                                   |
                      +------------+----------+
                      |    LoggingExtension   |
                      +-----------------------+
                      | initialize()          |
                      | consume_api()         |
                      | _handle_notification()|
                      +-----------------------+

Class design for LoggingDriver would look like::

                        +-----------------------------+
                        |    LoggingDriver            |
                        +-----------------------------+
                        |SUPPORTED_LOGGING_TYPES=set()|
                        |initialize()                 |
                        |start_logging(log_resource)  |
                        |stop_logging(log_resource)   |
                        +--------------^--------------+
                                       |
                      +----------------+-----------------+
                      |                                  |
       +--------------+---------------+   +--------------+---------------+
       | IptablesLoggingDriver        |   | OVSFirewallLoggingDriver     |
       +------------------------------+   +------------------------------+
       | SUPPORTED_LOGGING_TYPES=[...]|   | SUPPORTED_LOGGING_TYPES=[...]|
       | initialize()                 |   | initialize()                 |
       | start_logging(log_resource)  |   | start_logging(log_resource)  |
       | stop_logging(log_resource)   |   | stop_logging(log_resource)   |
       | _handle_logging()            |   |                              |
       | _create_security_group_log   |   +------------------------------+
       | _delete_security_group_log   |
       | ...                          |
       +------------------------------+


For OVS firewall: OVSFirewallLoggingDriver will act as a controller program.
It runs in each compute node to handle packet_in to produce a log line per
packet to a file.  To generate a packet in message for ACCEPT event: The
OVSFirewallLoggingDriver will insert "actions=controller" to flows matched
with security group rules whose conntrack state is NEW (first packet). It  will
also insert flows with 'actions=controller' before flows has 'actions=drop' and
conntrack state is INVALID or NOT ESTABLISHED to generate packet_in messages
for DROP event via port-number for project-isolation. For a flow example would
look like:

* For ingress tables::

    table=82, priority=73,ct_state=+new-est,icmp,reg5=0xa,dl_dst=fa:16:3e:1d:d0:8b actions=ct(commit,zone=NXM_NX_REG6[0..15]),strip_vlan,output:10,CONTROLLER:65535
    table=82, priority=70,ct_state=+new-est,icmp,reg5=0xa,dl_dst=fa:16:3e:1d:d0:8b actions=ct(commit,zone=NXM_NX_REG6[0..15]),strip_vlan,output:10
    table=82, priority=70,ct_state=+est-rel-rpl,icmp,reg5=0xa,dl_dst=fa:16:3e:1d:d0:8b actions=strip_vlan,output:10
    table=82, priority=53,ct_mark=0x1,reg5=0xa actions=CONTROLLER:65535
    table=82, priority=50,ct_mark=0x1,reg5=0xa actions=drop
    table=82, priority=50,ct_state=+inv+trk actions=drop
    table=82, priority=50,ct_state=+est-rel+rpl,ct_zone=1,ct_mark=0,reg5=0xa,dl_dst=fa:16:3e:1d:d0:8b actions=strip_vlan,output:10
    table=82, priority=50,ct_state=-new-est+rel-inv,ct_zone=1,ct_mark=0,reg5=0xa,dl_dst=fa:16:3e:1d:d0:8b actions=strip_vlan,output:10
    table=82, priority=43,ct_state=-est,reg5=0xa actions=CONTROLLER:65535
    table=82, priority=40,ct_state=-est,reg5=0xa actions=drop
    table=82, priority=40,ct_state=+est,ip,reg5=0xa actions=ct(commit,zone=NXM_NX_REG6[0..15],exec(load:0x1->NXM_NX_CT_MARK[]))
    table=82, priority=40,ct_state=+est,ipv6,reg5=0xa actions=ct(commit,zone=NXM_NX_REG6[0..15],exec(load:0x1->NXM_NX_CT_MARK[]))
    table=82, priority=0 actions=drop

* For egress tables::

    table=72, priority=73,ct_state=+new-est,icmp,reg5=0xa,dl_src=fa:16:3e:1d:d0:8b actions=resubmit(,73),CONTROLLER:65535
    table=72, priority=70,ct_state=+new-est,icmp,reg5=0xa,dl_src=fa:16:3e:1d:d0:8b actions=resubmit(,73)
    table=72, priority=70,ct_state=+est-rel-rpl,icmp,reg5=0xa,dl_src=fa:16:3e:1d:d0:8b actions=resubmit(,73)
    table=72, priority=53,ct_mark=0x1,reg5=0xa actions=CONTROLLER:65535
    table=72, priority=50,ct_mark=0x1,reg5=0xa actions=drop
    table=72, priority=50,ct_state=+inv+trk actions=drop
    table=72, priority=50,ct_state=+est-rel+rpl,ct_zone=1,ct_mark=0,reg5=0xa actions=NORMAL
    table=72, priority=50,ct_state=-new-est+rel-inv,ct_zone=1,ct_mark=0,reg5=0xa actions=NORMAL
    table=72, priority=43,ct_state=-est,reg5=0xa actions=CONTROLLER:65535
    table=72, priority=40,ct_state=-est,reg5=0xa actions=drop
    table=72, priority=40,ct_state=+est,ip,reg5=0xa actions=ct(commit,zone=NXM_NX_REG6[0..15],exec(load:0x1->NXM_NX_CT_MARK[]))
    table=72, priority=40,ct_state=+est,ipv6,reg5=0xa actions=ct(commit,zone=NXM_NX_REG6[0..15],exec(load:0x1->NXM_NX_CT_MARK[]))
    table=72, priority=0 actions=drop

The reference implementation can be found in [5]_.

For Iptables-based: IptablesLoggingDriver implementation is quite clear.
IptablesLoggingDriver just adds NFLOG rules to iptables [4]_.

Cloud deployers can choose **log_driver** for OVS native firewall driver or
iptables driver by specifying **ovs_fw_log** or **iptables_log**. They are also
able to configure **rate_limit** and **burst_limit** [8]_ to limit maximum
packets logging per second.  Log output file can be specified by option
**local_output_log_base**. These options are defined in
/etc/neutron/plugins/ml2/ml2_conf.ini::

    [log]
    # Driver for security group logging in the L2 agent (string value)
    # 'ovs_fw_log' or 'iptables_log'
    log_driver = ovs_fw_log

    # Maximum packets logging per second. (integer value, at least 100)
    rate_limit = 100

    # Maximum number of packets per rate_limit(integer value, at least 25)
    burst_limit = 25

    # Output logfile
    local_output_log_base = /var/log/syslog

To consume log-data, operators can:

(1) use third-party services such as Monasca service to consume log-data.
(2) access directly to 'local_output_log_base' to take log-data.

Expected API behavior
---------------------

This spec takes security groups logging for an example:

Operators can collect security events (ACCEPT/DROP or ALL (both ACCEPT and
DROP)) for some cases:

    (1) collect events related to a specific security group applied to all VMs
        by passing its security group ID to 'resource_id'.

    (2) collect events related to a specific security group applied to a
        specific VM by passing its security group ID to 'resource_id' and its
        bound Neutron port ID to 'target_id'.

    (3) collect events related to all security groups being applied to a
        specific VM by passing its Neutron port ID to 'target_id'.

    (4) collect events related to security groups in a project: in this case
        operators do not pass any value to 'resource_id' or 'target_id'.

More details, this API will log first packet which is matched with security
group rule for ACCEPT events. It will also log all packets which is unmatched
with any security group rules for DROP events.

Please note that, depends on above use cases, if a new VM or a new security
group or a new security group rule is launched, its related security events
will be collected, too. On the other hands, if a VM or a security group or a
security group rule is deleted, its related security events will be not logged.
However, security events related to the remains will still be logged.

Future work beyond this spec
----------------------------

* Extending logging for FWaaS v2.
* Exposing the logging API for projects (e.g by using RBAC).

API operation sample
--------------------

Take below scenario as an example:

1. Operator boots a number of VMs, the VMs' traffic is restricted
   by security-group-rules.

2. Check supported logging capabilities

    GET /v2.0/logging/loggable-resources

    .. code-block:: json

        {
            "loggable_resources":[{"type": "security_group"}]
        }


3. Create a **logging-resource** and define events related to security groups
   need collecting for this project.  For example, operator wants to collect
   all ACCEPT & DROP events related to all security groups applied in project
   demo (Read `Expected API behavior`_ section for more details)

    POST /v2.0/logging/logs

    .. code-block:: python

        {
            "log": {
                "name": "create_log_test1",
                "description": "Collecting all security events in project demo",
                "project_id": "8d4c70a21fed4aeba121a1a429ba0d04",
                "resource_type": "security_group",
                "event": "ALL"}
        }

    Response

    .. code-block:: python

        {
            "log": {
                "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c793",
                "project_id": "8d4c70a21fed4aeba121a1a429ba0d04",
                "name": "create_log_test1",
                "description": "Collecting all security events in project demo",
                "enabled": True,
                "resource_type": "security_group",
                "event": "ALL",
                "resource_id": None,
                "target_id": None}
        }

4. Operator use external services (e.g Monasca) to consume log-data.


Example logging format
----------------------

Logging output should include timestamp, action(ACCEPT/DROP) project ID,
logging-resource ID, VM port ID, source MAC, destination MAC,
source IP address, destination IP address, protocol, source L4 port and
destination L4 port.  Making the format configurable is out of the scope
though and the format can be defined in implementation phase.

For initial implementation would look like:

* log-data of an ACCEPT event::

    May 5 09:05:07 action=ACCEPT project_id=736672c700cd43e1bd321aeaf940365c
    log_resource_ids=['4522efdf-8d44-4e19-b237-64cafc49469b', '42332d89-df42-4588-a2bb-3ce50829ac51']
    vm_port=e0259ade-86de-482e-a717-f58258f7173f
    ethernet(dst='fa:16:3e:ec:36:32',ethertype=2048,src='fa:16:3e:50:aa:b5'),
    ipv4(csum=62071,dst='10.0.0.4',flags=2,header_length=5,identification=36638,offset=0,
    option=None,proto=6,src='172.24.4.10',tos=0,total_length=60,ttl=63,version=4),
    tcp(ack=0,bits=2,csum=15097,dst_port=80,offset=10,option=[TCPOptionMaximumSegmentSize(kind=2,length=4,max_seg_size=1460),
    TCPOptionSACKPermitted(kind=4,length=2), TCPOptionTimestamps(kind=8,length=10,ts_ecr=0,ts_val=196418896),
    TCPOptionNoOperation(kind=1,length=1), TCPOptionWindowScale(kind=3,length=3,shift_cnt=3)],
    seq=3284890090,src_port=47825,urgent=0,window_size=14600)

* log-data of a DROP event::

    May 5 09:05:07 action=DROP project_id=736672c700cd43e1bd321aeaf940365c
    log_resource_ids=['4522efdf-8d44-4e19-b237-64cafc49469b'] vm_port=e0259ade-86de-482e-a717-f58258f7173f
    ethernet(dst='fa:16:3e:ec:36:32',ethertype=2048,src='fa:16:3e:50:aa:b5'),
    ipv4(csum=62071,dst='10.0.0.4',flags=2,header_length=5,identification=36638,offset=0,
    option=None,proto=6,src='172.24.4.10',tos=0,total_length=60,ttl=63,version=4),
    tcp(ack=0,bits=2,csum=15097,dst_port=80,offset=10,option=[TCPOptionMaximumSegmentSize(kind=2,length=4,max_seg_size=1460),
    TCPOptionSACKPermitted(kind=4,length=2), TCPOptionTimestamps(kind=8,length=10,ts_ecr=0,ts_val=196418896),
    TCPOptionNoOperation(kind=1,length=1), TCPOptionWindowScale(kind=3,length=3,shift_cnt=3)],
    seq=3284890090,src_port=47825,urgent=0,window_size=14600)


Data Model Impact
-----------------

This model defines a generic model for all logging resources like security
groups and firewall.

Log model would look like:

+--------------+-------+-------+---------+-----------+-------------------------------------+
|Attribute     |Type   |Access |Default  |Validation/|Description                          |
|Name          |       |       |Value    |Conversion |                                     |
+==============+=======+=======+=========+===========+=====================================+
|id            |string |RO     |generated|uuid       |Identity                             |
|              |(UUID) |       |         |           |                                     |
+--------------+-------+-------+---------+-----------+-------------------------------------+
|project_id    |string |RW(No  |N/A      |uuid       |Project ID of Logging                |
|              |(UUID) |update)|         |           |object owner                         |
+--------------+-------+-------+---------+-----------+-------------------------------------+
|name          |string |RW     |N/A      |none       |Logging object name                  |
+--------------+-------+-------+---------+-----------+-------------------------------------+
|description   |string |RW     |N/A      |none       |Logging object description           |
+--------------+-------+-------+---------+-----------+-------------------------------------+
|enabled       |bool   |RW     |True     |Boolean    |Enable/disable log                   |
+--------------+-------+-------+---------+-----------+-------------------------------------+
|resource_type |string |RW(No  |N/A      |none       |Logging resource type, it could be   |
|              |       |update)|         |           |'security_group' in the initial      |
|              |       |       |         |           |implemenation                        |
+--------------+-------+-------+---------+-----------+-------------------------------------+
|resource_id   |string |RW(No  |N/A      |uuid       |ID of resource which is enabled to   |
|              |(UUID) |update)|         |           |be logged. When a resource_id (which |
|              |       |       |         |           |could be a security group ID in the  |
|              |       |       |         |           |initial implementation) is specified,|
|              |       |       |         |           |its related events will be collected.|
|              |       |       |         |           |For detail usage refer to            |
|              |       |       |         |           |`Expected API behavior`_             |
+--------------+-------+-------+---------+-----------+-------------------------------------+
|event         |string |RW(No  |N/A      |none       |ACCEPT/DROP or ALL can be specified. |
|              |       |update)|         |           |ALL is set as default.               |
+--------------+-------+-------+---------+-----------+-------------------------------------+
|target_id     |string |RW(No  |N/A      |uuid       |ID of resource (which could be a port|
|              |(UUID) |update)|         |           |ID in the initial implementation),   |
|              |       |       |         |           |where we want to collect log. When a |
|              |       |       |         |           |target_id is specified, events       |
|              |       |       |         |           |related to it will be colleted.      |
|              |       |       |         |           |For detail usage refer to            |
|              |       |       |         |           |`Expected API behavior`_             |
+--------------+-------+-------+---------+-----------+-------------------------------------+


REST API Impact
---------------
The new resources will be added.

.. code-block:: python

    LOGGING_PREFIX = '/logging'
    EVENTS = ['ACCEPT', 'DROP', 'ALL']
    RESOURCE_ATTRIBUTE_MAP = {
        'log': {
            'id': {'allow_post': False, 'allow_put': False,
                   'validate': {'type:uuid': None},
                   'is_visible': True, 'primary_key': True},
            'project_id': {'allow_post': True, 'allow_put': False,
                           'required_by_policy': True,
                           'validate': {'type:string': attr.TENANT_ID_MAX_LEN},
                           'is_visible': True},
            'name': {'allow_post': True, 'allow_put': True,
                     'validate': {'type:string': attr.NAME_MAX_LEN},
                     'default': '', 'is_visible': True},
            'description': {'allow_post': True, 'allow_put': True,
                            'validate': {'type:string': attr.DESCRIPTION_MAX_LEN},
                            'default': '', 'is_visible': True},
            'resource_type': {'allow_post': True, 'allow_put': False,
                              'required_by_policy': True,
                              'validate': {'type:validate_log_resource_type': None},
                              'is_visible': True},
            'resource_id': {'allow_post': True, 'allow_put': False,
                            'is_visible': True, 'default': None,
                            'validate': {'type:uuid_or_none': None}},
            'event': {'allow_post': True, 'allow_put': False,
                      'validate': {'type:values': EVENTS},
                      'is_visible': True, 'default': 'ALL'},
            'target_id': {'allow_post': True, 'allow_put': False,
                          'is_visible': True, 'default': None,
                          'validate': {'type:uuid_or_none': None}},
            'enabled': {'allow_post': True, 'allow_put': True,
                        'is_visible': True, 'default': True,
                        'convert_to': attr.convert_to_boolean},
        },

        'loggable_resources':{
            'type': {'allow_post': False, 'allow_put': False,
                     'is_visible': True}
        }
    }


+------------------+-------------------------------------------------+-------+
|Object            |URI                                              |Type   |
+==================+=================================================+=======+
|log               |/logging/logs                                    |POST   |
+------------------+-------------------------------------------------+-------+
|log               |/logging/logs                                    |GET    |
+------------------+-------------------------------------------------+-------+
|log               |/logging/logs/{logging-resource-id}              |GET    |
+------------------+-------------------------------------------------+-------+
|log               |/logging/logs/{logging-resource-id}              |DELETE |
+------------------+-------------------------------------------------+-------+
|log               |/logging/logs/{logging-resource-id}              |PUT    |
+------------------+-------------------------------------------------+-------+
|loggable-resource |/logging/loggable-resources                      |GET    |
+------------------+-------------------------------------------------+-------+


Security Impact
---------------

This REST API will only be accessible to **admin_only** which is controlled by
policy.json.

The volume of log highly depends on user traffic. It affects performance and
it is a potential security impact (a kind of DoS).


Notifications Impact
--------------------

None


Operators CLI Impact
--------------------

Additional methods will be added to neutron OSC plugin to create, update,
delete,
list and get logging resource.

For logging resource::

    openstack network logging create
                --resource-type <resource-type>
                [--description <description>]
                [--enable | --disable]
                [--resource-id <resource-id>]
                [--event=<ACCEPT/DROP/ALL>]
                [--target-id=<port-id>]
                [--project <project> [--project-domain <project-domain>]]
                name

    openstack network logging list

    openstack network logging set
                [--name <new-name>]
                [--description <description>]
                [--enabled |--disabled]
                <logging-resource-id>

    openstack network logging show <logging-resource-id>
    openstack network logging delete <logging-resource-id>

Check supported logging capabilities::

    openstack network loggable resources list

Performance Impact
------------------

Currently, logged data will be stored into files. So this may increase a little
bit performance overhead.

IPv6 Impact
-----------

Support both IPv4 and IPv6.


Other Deployer Impact
---------------------

Configure to enable logging API service.


Developer Impact
----------------

None


Community Impact
----------------

None


Alternatives
------------

Changing the security-group API by adding 'logging' attribute to Security
Group API resource. It would break the compatibility with the EC2 API [6]_.
And we should avoid changing FWaaS API by extend it [7]_.

Hence, the logging API allow enable logging is necessary.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  y-furukawa-2

Other contributors:
  hoangcx,
  annp,
  tuhv


Work Items
----------

* Finalize a way to log data
* Implement

  * REST API + API tests
  * Database model & database migrations
  * Oslo versioned object database implementation
  * Iptables based reference implementation
  * OSC support


Dependencies
============

None


Testing
=======

* Unit Test
* Functional test
* Fullstack test

Tempest Tests
-------------

None, since this is covered by the in-tree API tests.


Functional Tests
----------------

Regression test with the logging feature enabled.
Test if the logging feature actually works.


API Tests
---------

The new API interface will be tested via API tests, to ensure all the
operations work as expected.


Documentation Impact
====================

User Documentation
------------------

* Add some usages into the networking guide for operator.
* Write how to enable the logging API plugin.

References
==========

.. [1] https://review.openstack.org/#/c/426946/
.. [2] http://eavesdrop.openstack.org/meetings/neutron_drivers/2017/neutron_drivers.2017-04-27-22.03.log.html
.. [3] https://review.openstack.org/#/c/395504/
.. [4] https://review.openstack.org/#/c/445827/
.. [5] https://review.openstack.org/#/c/396104/
.. [6] https://review.openstack.org/132134
.. [7] https://review.openstack.org/132133
.. [8] http://openvswitch.org/support/dist-docs/ovs-vswitchd.conf.db.5.html
