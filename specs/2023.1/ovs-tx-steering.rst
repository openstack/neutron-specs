..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Expose backend hints in the port API, hint ovs-tx-steering
==========================================================

RFE: https://bugs.launchpad.net/neutron/+bug/1990842

Introduce a port attribute that allows passing in backend specific hints
mainly to allow backend specific performance tuning.

Problem Description
===================

Some of our performance sensitive users would like to tweak Open vSwitch's Tx
packet steering option [1]_ under OpenStack.

This is available since Open vSwitch v2.17.0. [2]_, [3]_

Proposed Change
===============

We propose to introduce two new extension: ``port-hints`` and
``port-hint-ovs-tx-steering``.

The purpose of decomposing this feature to two extensions is
extensibility: to allow other hints in the future. ``port-hints``
signals the presence of a new port attribute, for details see
below. ``port-hint-foobar`` signals what values of the new port attribute
(or which is the same: what hints) are known to a Neutron instance.

``port-hints`` adds a new port attribute: ``hints``.  Its default
policy is ``admin_only``.  Its value is a dict (by default empty).
The dict is keyed by the standard mechanism driver aliases as in [4]_.
This spec allows only a single key: ``openvswitch``. Other keys for
other mechanism drivers may be introduced by later specs.  The value
for a mechanism_driver is possibly a complex structure, but at least
a dict on the top level.  In this spec we only partially define its
format - introducing one hint. This is marked by the second extension
``port-hint-ovs-tx-steering``.  Definition of other hints is left to
future specs. Here consider this partial body for a create port request:

::

    {
        "port": {
            "hints": {
                "openvswitch": {"other_config": {"tx-steering": "thread"|"hash"}}}
            ...
    }

Everything in ``hints`` is interpreted not as a demand, but as a
suggestion. That is, neutron is free to ignore some or all of the
requested hints without returning an error response or putting the
port into an error status. Particularly neutron is free to ignore the
requested hints when the port is bound by a different mechanism driver.

``hints`` can be set at port create or update, but there's no guarantee
that an update to the hints after the port was bound will have any effect.

``hints`` is intentionally not named ``binding:hints`` and it should
not be affecting the binding process and decision.

In the ``openstack`` CLI we propose to expose the above API feature as:

::

    openstack port create/set --hint HINT-ALIAS[:HINT-VALUE] [--hint ...] ...

For example:

::

    openstack port create --hint ovs-tx-steering:thread ...
    openstack port create --hint ovs-tx-steering:hash ...

The above CLI syntax allows boolean and key:value type hints.  There is
an arbitrary but documented mapping between a HINT-ALIAS and the full
fledged data structure passed in the API.

Alternatively we could pass in a sizable JSON blob, but I believe that
would lead to a cumbersome CLI experience.

Given the ovs-tx-steering hint passed in, ovs-agent can set the
corresponding OVS interface's other_config (using the python native
interface of course, not ovs-vsctl):

::

    sudo ovs-vsctl set Interface ovs-interface-of-the-port other_config:tx-steering=thread
    sudo ovs-vsctl set Interface ovs-interface-of-the-port other_config:tx-steering=hash

The default value of hint ``ovs-tx-steering`` is ``thread``.

API Impact
----------

Add a new admin_only field to the port resource called ``hints``. This
field can be present in GET, POST and PUT requests. This field cannot be
longer than 4095 characters. For its semantics, please see above.

DB Impact
---------

Introduce a new table ``porthints`` and autojoin it with the ``ports``
table.

Client Impact
-------------

Relevant changes in osc and openstacksdk.

Testing
-------

* Unit tests.
* Tempest tests in neutron-tempest-plugin.

Assignee(s)
-----------

* Bence Romsics <bence.romsics@gmail.com>

References
==========

.. [1] https://docs.openvswitch.org/en/latest/topics/userspace-tx-steering/

.. [2] https://github.com/openvswitch/ovs/blob/7af5c33c1629b309cbcbe3b6c9c3bd6d3b4c0abf/NEWS#L103

.. [3] https://github.com/openvswitch/ovs/commit/c18e707b2f259438633af5b23df53e1409472871

.. [4] https://opendev.org/openstack/neutron/src/commit/31af8d1539c9c88be12c37120a6d43df528349bd/setup.cfg#L98-L112
