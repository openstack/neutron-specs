..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Extension for Network Security Group
=================================================

https://blueprints.launchpad.net/neutron/+spec/network-security-group

Problem description
===================

Neutron offers virtual network management functionality.
Since we can define any number of networks/subnets in a flexible way,
so virtual network topologies
tend to map security policy. On the other hand, we can specify security group
on a per port basis.
In this blueprint, we are proposing the addition of security_groups
as an extension attribute for the Network resource, for better API usability.
As an example, with this extension a tenant can define network-wide default security group(s),
so any vms that come up on any subnet in that network will be subject to the corresponding group rules.

Proposed change
===============

Adding security_groups attribute for network.
Filter operations are applied to Port security groups (as currently)
to which the Network security groups will be appended.
Please take a look a detailed example in data model impact.

Network security group and port security group will be union.
All the rules in relevant Network SGs and Port SGs should be evaluated in an arbitrary order
(since ordering doesn't matter here) with the implicit DROP ALL rule at the end.
That is akin to normal port SG evaluation with multiple SGs associated.

Alternatives
------------

We can automate process using Heat. However, this blue print simplifies
security group configuration management.

Data model impact
-----------------

In this blue print, we are adding security_groups attribute on network.
so security group apply model will be this model.

.. blockdiag::

  blockdiag model {
    network_security_group -> port_security_group -> vm;
  }


We will implement this using Binding table and lazy load.
We will discuss performance impact on this performance impact section.

::

    class SecurityGroupNetworkBinding(model_base.BASEV2):
        """Represents binding between neutron networks and security profiles."""

        networks_id = sa.Column(sa.String(36),
                            sa.ForeignKey("networks.id",
                                          ondelete='CASCADE'),
                            primary_key=True)
        security_group_id = sa.Column(sa.String(36),
                                      sa.ForeignKey("securitygroups.id"),
                                      primary_key=True)

        # Add a relationship to the Network model in order to instruct SQLAlchemy to
        # eagerly load security group bindings
        networks = orm.relationship(
            models_v2.Network,
            backref=orm.backref("security_groups",
                                lazy='joined', cascade='delete')
        # Add a relationship to the Port model in order to instruct SQLAlchemy to
        # eagerly load security group bindings
        ports = orm.relationship(
            models_v2.Port,
            backref=orm.backref('network_security_groups'),
            primaryjoin="Port.network_id==SecurityGroupNetworkBinding.network_id")


REST API impact
---------------

For REST API, we will add security_groups attributes as extended attributes
This extension also adds network_security_groups which will show the list
of associated networks security groups

::

    SECURITYGROUPS = 'security_groups'
    EXTENDED_ATTRIBUTES_2_0 = {
        'networks': {SECURITYGROUPS: {'allow_post': True,
                                   'allow_put': True,
                                   'is_visible': True,
                                   'convert_to': convert_to_uuid_list_or_none,
                                   'default': attr.ATTR_NOT_SPECIFIED}},
        'ports': {"network_%s" % SECURITYGROUPS: {'allow_post': False,
                                   'allow_put': False,
                                   'is_visible': True,
                                   'default': attr.ATTR_NOT_SPECIFIED}}}}


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

No

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

No. Network and SecurityGroup has quotas. We don't allow specify duplicated
security group_id


Notifications impact
--------------------

Update for security group network binding should be notified to the agent.

Other end user impact
---------------------

We will add new attributes for python-neutronclient.
- specifying security_groups on network
- show network_security_groups in show-ports

Performance Impact
------------------

- Notification event
    This change will introduce new notification, however this improves performance.
    So let's say there are 1000 port in a network. Let's say a user want to
    add new security group for the ports.
    The user should update 1000 port which will issue 1000 notification without
    this extension.This extension make it only one update.

- Agent side performance
    Implementation for realizing remote_group_id in security group is a kind of heavy operation.
    In current implementation, we are managing lists of IP addresses which belongs to one group.
    In network_security_group, there is no need to manage this because we can use subnet's
    prefix as a block.

- DB Performance
    This change will impact DB performance because this joins new binding
    tables. However, this extension can also improve performance such as same discussion
    with notification event. Actually, notification will issue DB calls.
    This extension may decrease number of DB call and rows in DB.
    One example is a default security groups. This will automatically apply all ports.
    If we do this in network_security_group, we can dramatically decrease security group
    related load.


Other deployer impact
---------------------

* Configuration options
  enable_network_security_group : boolean (Default False)
  Since this may impact DB performance, we let operator decide enable
  this function or not

  default_in_network_security_group : boolean (Default: False)
  If True, default security group will be applied for every network, and ports won't
  have default security group.

* Is this a change that takes immediate effect after its merged, or is it
  something that has to be explicitly enabled?

  Network security group functionality is default off until we see this is widely used.

* If this change is a new binary, how would it be deployed?

  N/A


Developer impact
----------------

Discuss things that will affect other developers working on OpenStack,
such as:

* If the blueprint proposes a change to the API, discussion of how other
  plugins would implement the feature is required.

I'm going to apply this for ML2 plug-in.

Implementation
==============

Assignee(s)
-----------

Nachi Ueno <nati-ueno>

Work Items
----------

- Implement extension
- Benchmark DB performance
- Update iptables based implementation
- Horizon support

Dependencies
============

* N/A

Testing
=======

- CRUD event for network_security_group
- show_port
- Tempest

Documentation Impact
====================

- User, admin doc should be updated by new REST API model and configuration parameters

References
==========

* N/A
