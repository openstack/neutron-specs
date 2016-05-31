..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
QoS improved validation mechanism for rules and port types
==========================================================

https://bugs.launchpad.net/neutron/+bug/1586056

While the QoS service models an abstraction of rules that can be applied
to port packets by different plugins, it's currently not able to correctly
handle the variety of different port types existing in a deployment,
or specific limitations of certain plugins or technologies.

Problem Description
===================

The administrator may desire, and it's something common,
to mix openvswitch and SR-IOV. While SR-IOV is used for higher throughput
functions, ovs is there for handling normal VM loads.

One of the first differences we will find in SR-IOV is that, it's not able
to limit bandwidth in both directions, while OVS and Linuxbridge are able.

Policy compatibility
--------------------

For example, the DSCP marking rule is currently implemented in OVS alone, this
means that a QoS policy applied to a linuxbridge or sr-iov bound port won't be
fully realized.

Rule compatibility
------------------

If we look at rule validation, we are planning on expanding existing rules
with new parameters [3], but in such cases some mechanisms declaring themselves as
supporters of such rule may not support the specific parameter (direction, traffic
classification, etc..). In the case of non-ml2 plugins, some types of ports may not
support specific parameters as well (based on the nature of the port).


Proposed Change
===============

We propose validation changes to policies, rules parameters, ports (qos-policy) and
networks (qos-policy) at the API level. As a result when a conflicting change
is detected, such conflict is immediately reported to the API caller, and the
operation is stopped with a 409 HTTP Conflict error::

  $ neutron qos-<type>-rule-update <pol-id> <rule-id> -<parameter> <value>
  Error: you have a port (<port id>) of type (..) attached to that rule,
         which is unable to accept <parameter>=<value>.


In the context of ML2, the mechanism drivers will define not only the rule types
they support, but also the parameter values, going from::

    supported_qos_rule_types = [qos_consts.RULE_TYPE_BANDWIDTH_LIMIT,
                                    qos_consts.RULE_TYPE_DSCP_MARK]

to::

   supported_qos_rule_types = {
           qos_consts.RULE_TYPE_BANDWIDTH_LIMIT: {
                       "max_kbps": qos_consts.ANY_VALUE,
                       "max_burst_kbps": qos_consts.ANY_VALUE,
                       "parameter": ["value1", "value2"],
                       "parameter2": qos.Parameter2Validator }
           qos_consts.RULE_TYPE_DSCP_MARK: {
                       "dscp_mark": qos_consts.ANY_VALUE}
    }


For validations which can't be handled by simple list, another validator class will be provided
to be subclassed which will require a validation and a description of how the validation is
accomplished (To make sure we have a verbose description in place if we wanted to report
capabilities in the future, or to provide more meaningful errors to users of the API).

A different mechanism driver not supporting the new "parameter" would have a declaration as
follows, where "parameter" is not declared in the dictionary::

   supported_qos_rule_types = {
           qos_consts.RULE_TYPE_BANDWIDTH_LIMIT: {
                       "max_kbps": qos_consts.ANY_VALUE,
                       "max_burst_kbps": qos_consts.ANY_VALUE,
           qos_consts.RULE_TYPE_DSCP_MARK: {
                       "dscp_mark": qos_consts.ANY_VALUE}
    }


Where Parameter2Validator example, and the generic class for validation would look like::

    class ParameterValidator(object):

        @abc.abstractmethod
        def validate(self, value):
            pass


    class Parameter2Validator(ParameterValidator):

        description = "Validates that parameter2 is higher than two"

        def validate(self, value):
            if value > 2:
                return
            raise Exception..(...)

Updating existing mechanism drivers to expose the new details will be necessary,
but we will provide one cycle with support for the old form, and deprecation notice
for existing out-of-tree mechanism drivers.

New functions will be added to the QoS service plugin "notification" interface::

  def validate_policy_for_port(self, context, policy, port):
    ...

  def validate_policy_for_network(self, context, policy, network_id):
    ...

For the second function, the base driver provides a default implementation
that will fetch the network ports and iterate over validate_policy_for_port,
but can be overriden by the implementation.

This will be responsible for verifying the policy/port pairs, and
throwing a detailed exception in case of a conflicting case.

In ML2, we propose to use the in process callbacks to validate the
BEFORE_CREATE/BEFORE_UPDATE of NETWORK and PORT objects. While the
modification of rules will be handled by the QoS plugin alone
consuming the validate_policy_for_port function that will be provided
by the implementation.

Notes
-----
QoS "notification" driver is certainly an unfortunate name for the interface and
may need to be changed into something more sensible in the future.

References
==========
[1] https://bugs.launchpad.net/neutron/+bug/1586056
[2] https://review.openstack.org/#/c/319694/
[3] https://bugs.launchpad.net/neutron/+bug/1560961
