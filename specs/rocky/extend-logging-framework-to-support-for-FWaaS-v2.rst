..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Extend logging framework to support FWaaS v2
================================================

https://bugs.launchpad.net/neutron/+bug/1720727

All content of this spec is based on `original spec <https://specs.openstack.org/openstack/neutron-specs/specs/pike/logging-API-for-security-group-rules.html>`_
and to extend supporting the logging feature for FWaaS v2 which was mentioned
in the *Future work beyond this spec* section.

We expect reviewers to check `original spec`_ before going ahead with below sections.

Problem Description
===================

The current logging framework just supports security group as an initial
implementation. FWaaS v2 lacks this useful feature.

Proposed Change
===============

To extend the feature to support FWaaS v2, we would like to propose
additional information comparable to *Proposed Change* section in
`original spec`_ as following:

Logging implementation
----------------------

Currently, Logging API is designed as a service plugin. It is also defined as a
generic logging API for resources such as security groups and firewall.
Reference implementation can be found in [1]_.

The server side is one aspect we need to handle. At the moment in Neutron
Security Group logging implementation, we have a function as request validator [2]_.
This code should be moved out from neutron to neutron-lib and will
be more generic so it can handle both Neutron SG and FWaaS.
The expected outcome of this step is to make sure this validator is still
working for Security Group and it can also be applied for FWaaS.


Regarding to agent side:

For L2 layer, the *LoggingExtension* is an L2 agent extension. It is common for
all logging resources like security groups and firewall. For L3 layer,
*LoggingL3AgentExtension* is an L3 agent extension dedicated for firewall.
These agent extensions will receive CREATED/UPDATED/DELETED events of the
logging resource and pass these events to logging drivers. Each logging driver
defines the resources it supports. In case of FWaaS v2, logging driver named
*FWaaSv2LoggingDriver* will be implemented. This class will inherit
from *LoggingDriver* class.

Regarding to driver side:

There are two drivers will be implemented in order to support for both
L2 layer (instance) and L3 layer (router).

For L2 layer, a driver named *FWaaSv2L2LoggingDriver* will be implemented. It
acts as a controller program.
This driver will insert flows log into table=91 and table=92 with
ct_state=NEW to generate ACCEPT events, insert flows log into
table=93 to generate DROP events.

For L3 layer, a driver named *FWaaSv2L3LoggingDriver* will be implemented based on
iptables in user namespace level. It runs in network node to handle router's
port by adding NFLOG rules to iptables. We would like to propose detail solution
as following:

(1) The structure of iptables rules.

    We will introduce two new chains:

    - neutron-l3-agent-accepted: log first accept packet and accept all
      packets which are matched with firewall rules.

    - neutron-l3-agent-dropped: log and drop all packets.


    New iptables structure when logging is enabled would look like::

        Chain INPUT (policy ACCEPT)
        target     prot opt source               destination
        neutron-l3-agent-INPUT  all  --  anywhere             anywhere

        Chain FORWARD (policy ACCEPT)
        target     prot opt source               destination
        neutron-filter-top  all  --  anywhere             anywhere
        neutron-l3-agent-FORWARD  all  --  anywhere             anywhere

        Chain OUTPUT (policy ACCEPT)
        target     prot opt source               destination
        neutron-filter-top  all  --  anywhere             anywhere
        neutron-l3-agent-OUTPUT  all  --  anywhere             anywhere

        Chain neutron-filter-top (2 references)
        target     prot opt source               destination
        neutron-l3-agent-local  all  --  anywhere             anywhere

        Chain neutron-l3-agent-FORWARD (1 references)
        target     prot opt source               destination
        neutron-l3-agent-scope  all  --  anywhere             anywhere
        neutron-l3-agent-iv4dd529723  all  --  anywhere             anywhere
        neutron-l3-agent-ov4dd529723  all  --  anywhere             anywhere
        neutron-l3-agent-fwaas-defau  all  --  anywhere             anywhere
        neutron-l3-agent-fwaas-defau  all  --  anywhere             anywhere

        Chain neutron-l3-agent-INPUT (1 references)
        target     prot opt source               destination
        ACCEPT     all  --  anywhere             anywhere             mark match 0x1/0xffff
        DROP       tcp  --  anywhere             anywhere             tcp dpt:9697

        Chain neutron-l3-agent-OUTPUT (1 references)
        target     prot opt source               destination

        Chain neutron-l3-agent-fwaas-defau (2 references)
        target     prot opt source               destination
        DROP       all  --  anywhere             anywhere

        Chain neutron-l3-agent-iv4dd529723 (1 references)
        target     prot opt source               destination
        neutron-l3-agent-dropped       all  --  anywhere             anywhere             state INVALID
        neutron-l3-agent-accepted     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
        neutron-l3-agent-dropped       all  --  anywhere             anywhere

        Chain neutron-l3-agent-local (1 references)
        target     prot opt source               destination

        Chain neutron-l3-agent-ov4dd529723 (1 references)
        target     prot opt source               destination
        neutron-l3-agent-dropped       all  --  anywhere             anywhere             state INVALID
        neutron-l3-agent-accepted     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
        neutron-l3-agent-accepted     all  --  anywhere             anywhere

        Chain neutron-l3-agent-scope (1 references)
        target     prot opt source               destination
        DROP       all  --  anywhere             anywhere             mark match ! 0x4000000/0xffff0000

        Chain neutron-l3-agent-fw-chain (2 references)
        target     prot opt source               destination
        ACCEPT     all  --  anywhere             anywhere

        chain neutron-l3-agent-accepted
        target     prot opt source               destination
        NFLOG      all  --  anywhere             anywhere             state NEW limit: avg 100/sec burst 25 nflog-prefix  12823226497704342389
        ACCEPT     all  --  anywhere             anywhere

        chain neutron-l3-agent-dropped
        target     prot opt source               destination
        NFLOG      all  --  anywhere             anywhere             limit: avg 100/sec burst 25 nflog-prefix  12823226497704342389
        DROP       all  --  anywhere             anywhere


(2) How to capture packets and parse information of packets. This requires at least two steps.

    - First we need to dump packets into raw format.

      * We propose to implement a python binding for `libnetfilter_log`, same
        idea like [3]_.

    - After we have packets in raw format, we need to parse these data into
      human readable format.

      * In order to do that, we propose to use `ryu` [4]_ library for this step.
        PoC implementation looks like [5]_.

About how to configure logging feature,
See `networking guide <https://review.openstack.org/#/c/480117/>`_ for detail.

Expected API behavior
---------------------

This spec takes FWaaS v2 logging as an example:

Operators can collect security events (ALLOW/DROP or ALL) for some cases:

(1) Collect events related to a specific firewall group applied to all
    instances/routers ports by passing its firewall group ID to ``resource_id``.

(2) Collect events related to a specific firewall group applied to a
    specific instance/router by passing its firewall group ID to ``resource_id``
    and its bound Neutron port ID to ``target_id``.

(3) Collect events related to all firewall groups being applied to a
    specific instance by passing its Neutron port ID to ``target_id``.

(4) Collect events related to firewall groups in a project: in this case
    operators do not pass any value to ``resource_id`` or ``target_id``.


API operation sample
--------------------

Same as `original spec`_


Data Model Impact
-----------------

None

REST API Impact
---------------

Same as `original spec`_

Add support firewall_group as ``loggable_resource`` type.
In order to do that, there are two changes required:

- Make request validator and rpc callback to be more generic in order to
  support firewall_group.
- On FWaaS, use above generic methods to register to Neutron side.

Security Impact
---------------

Same as `original spec`_


Notifications Impact
--------------------

None


Operators CLI Impact
--------------------

Add `firewall_group` to be on of --resource-type. Also add `firewall_group`
in output of supported logging capabilities.

Performance Impact
------------------

Same as `original spec`_

IPv6 Impact
-----------

Same as `original spec`_


Other Deployer Impact
---------------------

None as it done along with logging feature.


Developer Impact
----------------

None


Community Impact
----------------

None


Alternatives
------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  y-furukawa-2

Other contributors:
  hoangcx,
  annp,
  cuongnv


Work Items
----------

* Finalize a way to log data
* Implement *FWaaSv2LoggingDriver* based reference implementation


Dependencies
============

None


Testing
=======

Same as `original spec`_

Tempest Tests
-------------

Same as `original spec`_


Functional Tests
----------------

Same as `original spec`_


API Tests
---------

Same as `original spec`_


Documentation Impact
====================

User Documentation
------------------

Same as `original spec`_

References
==========

.. [1] https://review.openstack.org/#/c/395504/
.. [2] https://github.com/openstack/neutron/blob/139c8341f4eaa5f214050d4f7f1cca3f2a1cae34/neutron/services/logapi/common/validators.py#L111
.. [3] https://github.com/commonism/python-libnetfilter
.. [4] https://github.com/osrg/ryu
.. [5] https://review.openstack.org/#/c/445827/26/neutron/privileged/agent/linux/libnetfilter_log.py
