..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
BFD as a Service for Neutron
============================

https://bugs.launchpad.net/neutron/+bug/1907089

Add possibility to define BFD monitors in Neutron and associate it with routes.


Problem Description
===================

BFD (Bidirectional Forwarding Detection) can be used to detect link failures
between a Neutron router and an arbitrary destination, like an external
non-Neutron router.

BFD can be used to provide quick, low-overhead link failure detection between
adjacent forwarding engines ([1]_) to check for example if an extra route
(a nexthop-destination pair) is alive or not, and change routes accordingly.

.. note::

    This spec is not covering other use cases like tunnel monitoring and HA or
    DVR router monitoring, or BGP or other routing protocol monitoring.


Proposed Change
===============

The goal of this spec is to add APIs to manage bfd_monitor instances, fetch
their status, associate a monitor to an extra route. The goal is to cover
the use case when Neutron managed bfd_monitor instances take the ``ACTIVE``
role, thus send BFD control packets, (see [2]_).

This gives possibility to the operators to monitor the routes
(destination - nexthop pairs) added for a router.

Monitoring scenarios
--------------------

* Two different nexthops for the same destination, like ecmp route legs

+------+----------+----------+------+
| dst1 | nexthop1 | bfd_dst1 | bfd1 |
+------+----------+----------+------+
| dst1 | nexthop2 | bfd_dst1 | bfd2 |
+------+----------+----------+------+

::

         ____nexthop1 ____
                          \ ____dst1
         ____nexthop2____ /


* monitoring 2 different destinations through the same nexthop, the case
  when bfd3 and bfd4 dst_ip is the same:

+------+----------+----------+------+
| dst3 | nexthop3 | bfd_dst3 | bfd3 |
+------+----------+----------+------+
| dst4 | nexthop3 | bfd_dst4 | bfd4 |
+------+----------+----------+------+

::

                           ____dst3
        _____nexthop3_____/
                          \____dst4


REST API Impact
---------------

New bfd monitor API
~~~~~~~~~~~~~~~~~~~

* Create bfd_monitor::

        POST /v2.0/bfd_monitors
        {
            "bfd_monitor": {
                "name": "bfd1",
                "mode": "asynchronous",
                "min_rx": 1000,
                "min_tx": 100,
                "multiplier": 3,
                "dst_ip": "10.0.0.13",
                "auth_type": "",
                "auth_keys": {1: "secret_1", 2: "secret_2"},
            }
        }

* List bfd_monitors::

        GET /v2.0/bfd_monitors

* Show bfd_monitor::

        GET /v2.0/bfd_monitors/{monitor_id}
        {
            "bfd_monitor": {
                "name": "bfd1",
                "project_id": "{project_id}",
                "description": "",
                "mode": "asynchronous",
                "min_rx": 1000,
                "min_tx": 100,
                "multiplier": 3,
                "dst_ip": "10.0.0.13",
                "src_ip": "169.254.112.144",
                "auth_type": "",
                "auth_keys": {1: "secret_1"},
            }
        }

* Update a bfd_monitor::

        PUT /v2.0/bfd_monitors/{monitor_id}
        {
            "bfd_monitor": {
                "name": "bfd_monitor1"
                "description": "foo",
                "min_rx": 2000,
                "min_tx": 200,
                "multiplier": 4
            }
        }

* Delete bfd_monitor::

        DELETE /v2.0/bfd_monitors/{monitor_id}

.. Note::

   Only an unassociated bfd_monitor can be deleted.

API fields and descriptions for bfd_monitor:

+-------------------+---------+-------+------+---------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                           |
+===================+=========+=======+======+=======================================+
| id                | uuid-str| No    | R    | id of bfd_monitor                     |
+-------------------+---------+-------+------+---------------------------------------+
| name              | String  | No    | CRU  | Human readable name for the           |
|                   |         |       |      | bfd_monitor (255 characters limit).   |
|                   |         |       |      | Does not have to be unique.           |
+-------------------+---------+-------+------+---------------------------------------+
| description       | String  | No    | CRU  | Human readable description for the    |
|                   |         |       |      | bfd_monitor (255 characters limit).   |
+-------------------+---------+-------+------+---------------------------------------+
| project_id        | String  | No    | R    | Owner of the bfd_monitor.             |
+-------------------+---------+-------+------+---------------------------------------+
| mode              | String  | No    | CR   | Can be ``asynchronous`` (default      |
|                   |         |       |      | common echo mode of BFD) or           |
|                   |         |       |      | ``demand`` (some other mechanism is   |
|                   |         |       |      | used to detect link state) and can    |
|                   |         |       |      | accept future modes like              |
|                   |         |       |      | ``one-arm-echo`` see [4]_ .           |
+-------------------+---------+-------+------+---------------------------------------+
| dst_ip            | String  | Yes   | CR   | The destination IP address to be      |
|                   |         |       |      | monitored. In case of singlehop bfd   |
|                   |         |       |      | this is the nexthop ip of the route,  |
|                   |         |       |      | for the general case (like multihop   |
|                   |         |       |      | bfd) this is an arbitrary IP (IPv4 or |
|                   |         |       |      | Ipv6) that can serve as BFD neighbor. |
+-------------------+---------+-------+------+---------------------------------------+
| src_ip            | String  | No    | CR   | IP address used as source for         |
|                   |         |       |      | transmitted BFD packets. An optional  |
|                   |         |       |      | field, if not specified, the IP set   |
|                   |         |       |      | on the interface, like on qg-xyz.     |
|                   |         |       |      | This can be set on the remote end to  |
|                   |         |       |      | be configured.                        |
+-------------------+---------+-------+------+---------------------------------------+
| min_rx            | Integer | No    | CRU  | The shortest interval, in millisecs,  |
|                   |         |       |      | at which this BFD session offers to   |
|                   |         |       |      | receive BFD control messages. At      |
|                   |         |       |      | least 1. Defaults is 1000.            |
+-------------------+---------+-------+------+---------------------------------------+
| min_tx            | Integer | No    | CRU  | The shortest interval, in millisecs,  |
|                   |         |       |      | at which this BFD session is          |
|                   |         |       |      | willing to transmit BFD control       |
|                   |         |       |      | messages. At least 1. Default is 100. |
+-------------------+---------+-------+------+---------------------------------------+
| multiplier        | Integer | No    | CRU  | The BFD detection multiplier, An      |
|                   |         |       |      | endpoint signals  a connectivity      |
|                   |         |       |      | fault if the given number of          |
|                   |         |       |      | consecutive BFD control messages fail |
|                   |         |       |      | to arrive. Default is 3.              |
+-------------------+---------+-------+------+---------------------------------------+
| status            | String  | N/A   | R    | Shows if the BFD monitor was          |
|                   |         |       |      | succesfully created in the backend,   |
|                   |         |       |      | but nothing about the session status, |
|                   |         |       |      | for that the session_status API       |
|                   |         |       |      | endpoint can be used.                 |
+-------------------+---------+-------+------+---------------------------------------+
| auth_type         | String  | No    | CR   | The Authentication Type, which can be |
|                   |         |       |      | ``password``, ``MD5``,                |
|                   |         |       |      | ``MeticulousMD5``, ``SHA1``,          |
|                   |         |       |      | ``MeticulousSHA1``, if empty no       |
|                   |         |       |      | authentication is used.               |
+-------------------+---------+-------+------+---------------------------------------+
| auth_key          | Dict    | No    | CR   | A dictionary of authentication key    |
|                   |         |       |      | chain in which key is an integer of   |
|                   |         |       |      | ``Auth Key ID`` and value is a string |
|                   |         |       |      | of ``Password`` or ``Auth Key``.      |
+-------------------+---------+-------+------+---------------------------------------+

.. Note::

    For using authentication with BFD, please check [5]_

API extension proposal for bfd_monitors:

::

    ALIAS = 'bfd-monitor'
    IS_SHIM_EXTENSION = False
    IS_STANDARD_ATTR_EXTENSION = False
    NAME = 'BFD monitors for Neutron'
    DESCRIPTION = "Provides support for BFD monitors"
    UPDATED_TIMESTAMP = "2021-02-12T11:00:00-00:00"
    BFD_MONITOR = 'bfd_monitor'
    BFD_MONITORS = 'bfd_monitors'
    BFD_SESSION_STATUS = 'bfd_session_status'

    BFD_MODE_ASYNC = 'asynchronous'
    BFD_MODE_DEMAND = 'demand'
    BFD_MODE_ONE_ARM = 'one_arm_echo'

    RESOURCE_ATTRIBUTE_MAP = {
        BFD_MONITORS: {
            'id': {'allow_post': False, 'allow_put': False,
                   'validate': {'type:uuid': None},
                   'is_visible': True,
                   'primary_key': True,
                   'enforce_policy': True},
            'name': {'allow_post': True, 'allow_put': True,
                     'validate': {'type:string': db_const.NAME_FIELD_SIZE},
                     'default': '', 'is_filter': True, 'is_sort_key': True,
                     'is_visible': True},
            'description': {'allow_post': True, 'allow_put': True,
                            'is_visible': True, 'default': '',
                            'validate': {
                                'type:string': db_const.DESCRIPTION_FIELD_SIZE}},
            'project_id': {'allow_post': True, 'allow_put': False,
                           'validate': {
                               'type:string': db_const.PROJECT_ID_FIELD_SIZE},
                           'required_by_policy': True,
                           'is_visible': True, 'enforce_policy': True},
            'mode': {'allow_post': True, 'allow_put': False,
                     'validate': {'type:string': db_const.STATUS_FIELD_SIZE},
                     'default': BFD_MODE_ASYNC, 'is_filter': True,
                     'is_sort_key': True, 'is_visible': True},
            'dst_ip': {'allow_post': True, 'allow_put': False,
                       'validate': {'type:ip_address': None},
                       'is_sort_key': True, 'is_filter': True,
                       'is_visible': True, 'default': None,
                       'enforce_policy': True},
            'src_ip': {'allow_post': True, 'allow_put': False,
                   'validate': {'type:ip_address_or_none': None},
                   'is_sort_key': True, 'is_filter': True,
                   'is_visible': True, 'default': None,
                   'enforce_policy': True},
            'min_rx': {'allow_post': True, 'allow_put': True,
                       'validate': {'type:non_negative': None},
                       'convert_to': converters.convert_to_int,
                       'default': 1000,
                       'is_visible': True, 'enforce_policy': True},
            'min_tx': {'allow_post': True, 'allow_put': True,
                       'validate': {'type:non_negative': None},
                       'convert_to': converters.convert_to_int,
                       'default': 100,
                       'is_visible': True, 'enforce_policy': True},
            'multiplier': {'allow_post': True, 'allow_put': True,
                           'validate': {'type:non_negative': None},
                           'convert_to': converters.convert_to_int,
                           'default': 3,
                           'is_visible': True, 'enforce_policy': True},
            'status': {'allow_post': False, 'allow_put': False,
                       'is_filter': True, 'is_sort_key': True,
                       'is_visible': True},
            'auth_type': {'allow_post': True, 'allow_put': False,
                          'validate': {'type:string_or_none':
                                       db_const.NAME_FIELD_SIZE},
                          'default': constants.ATTR_NOT_SPECIFIED,
                          'is_visible': True},
            'auth_key': {'allow_post': True, 'allow_put': False,
                         'validate': {'type:dict_or_none': None},
                         'default': constants.ATTR_NOT_SPECIFIED,
                         'is_visible': True},
        },
    }
    SUB_RESOURCE_ATTRIBUTE_MAP = {}
    ACTION_MAP = {
        BFD_MONITOR: {
            'bfd_session_status': 'GET',
            'bfd_monitor_associations': 'GET',
        }
    }
    ACTION_STATUS = {}
    REQUIRED_EXTENSIONS = [l3.ALIAS]
    OPTIONAL_EXTENSIONS = []

* Show bfd session status (For details see rfc5880, State Variables section)::

        GET /v2.0/bfd_monitors/{monitor_id}/session_status
        {
            "bfd_session_status": {
                "remotes": [{
                    "type": "extra_route",
                    "router": {router_id},
                    "extra_route": {
                        "destination": "10.0.3.0/24",
                        "nexthop": "10.0.0.1"
                    },
                    "src_ip": "169.254.112.144",
                    "status": {
                        "SessionState": "Up",
                        "RemoteSessionState": "Up",
                        "LocalDiagnostic": "",
                        "RemoteDiagnostic": "",
                        "LocalDiscriminator": "",
                        "RemoteDiscriminator": "",
                        "Forwarding": "",
                    }
                },
                {
                    "type": "extra_route",
                    ...
                }]
            }
        }

* Show bfd_monitor associations::

       GET /v2.0/bfd_monitors/{monitor_id}/bfd_monitor_associations
       {
            "bfd_monitor_associations": {
                "remotes": [{
                    "type": "extra_route",
                    "router": {router_id},
                    "extra_route": {
                        "destination": "10.0.3.0/24",
                        "nexthop": "10.0.0.1"
                    },
                    "src_ip": "169.254.112.144",
                },
                {
                    "type": "extra_route",
                    ...
                }]
            }
       }


.. note::

    The current proposal is only for "type": "extra_route", so the
    extra_route's details (nexthop, destination as of now) will appear.

.. note::

    One bfd_monitor instance can be associated to several extra_routes.

.. note::

    ``src_ip`` is the IP address used as source for transmitted BFD packets.
    A read only field that practically shows the IP set on the interface,
    like on qg-xyz. This can be set on the remote end.

Short description of the status fields:

+--------------------+-----------------------------------------------------+
| Attribute          | Description                                         |
+====================+=====================================================+
| SessionState       | The state of the BFD session, it can be             |
|                    | ``admin_down``, ``down``, ``init``,  or ``up``.     |
+--------------------+-----------------------------------------------------+
| RemoteSessionState | The state of the remote endpoint's BFD session, it  |
|                    | can be ``admin_down``, ``down``, ``init``,  or      |
|                    | ``up``.                                             |
+--------------------+-----------------------------------------------------+
| LocalDiagnostic    | A diagnostic code specifying the local system's     |
|                    | reason for the last change in session state. The    |
|                    | error messages are defined  in section 4.1 of the   |
|                    | RFC 5880 (see [3]_).                                |
+--------------------+-----------------------------------------------------+
| RemoteDiagnostic   | A diagnostic code specifying the remote system's    |
|                    | reason for  the last  change in session state. The  |
|                    | error messages are defined  in section 4.1 of the   |
|                    | RFC 5880 (see [3]_).                                |
+--------------------+-----------------------------------------------------+
| LocalDiscriminator | The local discriminator for this BFD session, used  |
|                    | to uniquely identify it.                            |
+--------------------+-----------------------------------------------------+
| RemoteDiscriminator| The remote discriminator for this BFD session. This |
|                    | is the discriminator chosen by the remote system.   |
+--------------------+-----------------------------------------------------+
| Forwarding         | Reports whether the BFD session believes this       |
|                    | Interface  may  be used  to forward traffic. It can |
|                    | be ``true`` or ``false``.                           |
+--------------------+-----------------------------------------------------+

.. note::

   The above fields are optional, so based on the backend not all will be
   filled, and in worst case a cumulative ``Forwarding`` result will only
   present.

Changes to Router extra routes API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Changes to add or remove extra routes to router or Update router API::

        PUT /v2.0/routers/{router_id}
        {
            "router": {
                "routes": [
                    {
                        "destination": "179.24.1.0/24",
                        "nexthop": "172.24.3.99"
                        "bfd_monitor": {bfd_monitor_uuid}
                    },
                ]
            }
        }

        PUT /v2.0/routers/{router_id}/add_extraroutes
        {
            "router" : {
                "routes" : [
                   { "destination" : "10.0.3.0/24", "nexthop" : "10.0.0.13", "bfd_monitor": {bfd_monitor_uuid} },
                   { "destination" : "10.0.4.0/24", "nexthop" : "10.0.0.14" }
                ]
            }
        }

        PUT /v2.0/routers/{router_id}/remove_extraroutes
        {
            "router" : {
                "routes" : [
                   { "destination" : "10.0.3.0/24", "nexthop" : "10.0.0.13", "bfd_monitor": {bfd_monitor_uuid} }
                ]
            }
        }

.. note::

   The API will allow to remove bfd_monitor association from a route without
   deleting and recreating the whole nexthop-destination tuple.

* Get routes status::

        GET /v2.0/routers/{router_id}/routes_status
        {
            "router": {
                "routes": [
                    { "destination" : "10.0.3.0/24",
                      "nexthop" : "10.0.0.13",
                      "status": "UP",
                      "bfd_monitor": {bfd_monitor_uuid}
                    },
                ]
            }
        }

API extension proposal for allowing bfd_monitor association to extra_routes:

::

    ALIAS = 'bfd-for-extraroutes'
    IS_SHIM_EXTENSION = False
    IS_STANDARD_ATTR_EXTENSION = False
    NAME = 'BFD monitors for extraroutes'
    DESCRIPTION = ('Provides the possibility to associate a bfd_monitor to'
                   'extra_routes')
    UPDATED_TIMESTAMP = '2021-01-29T00:00:00+00:00'
    RESOURCE_NAME = l3.ROUTER
    COLLECTION_NAME = l3.ROUTERS
    ROUTES = 'routes'
    RESOURCE_ATTRIBUTE_MAP = {
        COLLECTION_NAME: {
            ROUTES: {
                'allow_post': False, 'allow_put': True,
                'validate': {'type:hostroutes': ['destination', 'nexthop',
                                                 'bfd_monitor_id']},
                'convert_to': converters.convert_none_to_empty_list,
                'is_visible': True,
                'default': constants.ATTR_NOT_SPECIFIED},
        }
    }
    SUB_RESOURCE_ATTRIBUTE_MAP = None
    ACTION_MAP = {}
    REQUIRED_EXTENSIONS = [l3.ALIAS, extraroute.ALIAS]
    OPTIONAL_EXTENSIONS = []
    ACTION_STATUS = {}

.. note::

    [6]_ proposes the change of the ``hostroutes`` validator to accept a list
    of possible fields to validate, like: ``destination``, ``nexthop`` and
    ``bfd_monitor_id``.

BFD itself and fetching bfd_monitor session status information can be quite
resource intensive operation (the information must be fetched from the
backend), so new API endpoint is proposed to fetch the status of monitoring
to not overload or affect ordinary router API operations.

The routes_status API endpoint gives feedback to the user if the monitored
link is up.

If the bfd_monitor's session_status is DOWN (BFD detects the given link dead)
the given route should be removed in the backend (on the command line: ip r
delete....), and set back if the monitor is UP again.

The bfd_monitor instance in the backend is created when the bfd_monitor is
associated with any routes, the ``status`` of the bfd_monitor is set to
``ACTIVE`` from ``DOWN``. The bfd_monitor's ``status`` set to ``DOWN`` when
the extra route updated and bfd_monitor value is set to empty, so the
bfd_monitor is deleted in the backend.

As a bfd_monitor can be associated to many routes, if it is associated to
at least one route and the creation in the backend is successful the
``status`` field of the bfd_monitor is ``ACTIVE``, and it will change to
``DOWN`` when the last route association is deleted (the bfd_monitor is
deleted from the last extra_route).

Backend
~~~~~~~

OVS can handle BFD on an interface and check the status of it.
The problem with it is that ovs can have 1 BFD session per port, and
to manage it ovsdb is the simplest way, but as LIU Yulong stated in [7]_
touching ovsdb from l3-agent can be weird.

A working solution with OVS is to create a BFD bridge, like br-bfd, and
add veth ports to it, and enable BFD on those interfaces.

::

          --------
         | br-int |
          --------
            | veth-xy-br-int
            |
            | veth-xy
          --------
         | br-bfd |
          --------

Another possibility is to use os-ken, but during my experiments the BFD
implementation in os-ken is not mature enough, and it needs ovsdb connection
anyway, so I do not suggest os-ken to be used for BFD.

Data Model Impact
-----------------

New table is created for bfd_monitors, routers table is changed to make it
possible to associate a bfd_monitor to an extra route (a ``nexthop`` -
``destination`` pair).

Security Impact
---------------

None


Performance Impact
------------------

BFD and fetching bfd_monitor status information can be quite resource intensive
operation as it can be fetched from the backend, it must be carefully
documented.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Lajos Katona <katonalala@gmail.com> (IRC: lajoskatona)

Work Items
----------

* REST API update.

* DB schema update.

* L3 agent update to handle BFD:

  * RPC change to send BFD data to l3 agent.

  * RPC change to fetch BFD status information from l3 agent.

* CLI update.

* Documentation.

* Tests and CI related changes.

Testing
=======

* Unit Test
* Functional test
* API test

Perhaps this is something that can be easily tested end-to-end with fullstack
tests. Need more investigation.


Documentation Impact
====================

User Documentation
------------------

New API and changes to legacy APIs like routers must be documented.


References
==========

.. [1] https://tools.ietf.org/html/rfc5880
.. [2] https://tools.ietf.org/html/rfc5880#section-6.1
.. [3] https://tools.ietf.org/html/rfc5880#section-6.8.1
.. [4] https://tools.ietf.org/html/draft-wang-bfd-one-arm-use-case-00
.. [5] https://tools.ietf.org/html/rfc5880#section-6.7
.. [6] https://review.opendev.org/c/openstack/neutron-lib/+/778859
.. [7] https://review.opendev.org/c/openstack/neutron-specs/+/767337/7/specs/wallaby/bfd_support.rst#377
