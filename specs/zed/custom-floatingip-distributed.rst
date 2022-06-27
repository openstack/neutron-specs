..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Floating IP distributed
=======================

Launchpad Bug:
https://bugs.launchpad.net/neutron/+bug/1978039

Neutron adds distributed attributes to each Floating IP. Users can set this
attribute according to their actual environment and use requirements.

Problem Description
===================

Neutron supports setting the floating IP to distributed. External traffic
can go directly to the compute node without passing through the network
node, and increases the network performance. This is very useful for the
demand for high-performance networks.

However, we can decide whether floating ips are distributed or centralized by
setting configuration option(ovn backend) or setting router's "distributed"
attribute(ovs dvr mode). This is global. After configuration, all floating
ips are centralized or distributed.

The actual use may be complicated:

* Not all vms with floating ip may require high-performance networks, and may
  only be exposed to the outside world to provide services.
* Due to equipment conditions, some compute nodes are not equipped with
  external network card. If enable distributed, the vm of this compute node
  cannot allocate floating ip.

Proposed Change
===============

Add new API extension to extend ``floatingip`` resource in Neutron with
``distributed`` attribute.
If not set this attribute, the ``distributed`` attribute of the router
will be used for the floating ip [1]_.

Server side changes
-------------------

A new API extension of Neutron will be added with new attribute for the
``floatingip`` resource. This new attribute will be called ``distributed``:

::

    RESOURCE_ATTRIBUTE_MAP = {
        l3.FLOATINGIPS: {
            "distributed": {
                'allow_post': True,
                'allow_put': True,
                'convert_to': converters.convert_to_boolean_if_not_none,
                'default': constants.ATTR_NOT_SPECIFIED,
                'is_visible': True,
                'is_filter': True
            }
        }
    }

DB Impact
---------

Extend the ``floatingips`` table with boolean column ``distributed``.

REST API Impact
---------------

New API extension: ``floatingip-distributed`` introducing new floatingip
attribute: ``distributed``.

Client Impact
-------------

Relevant changes in OSC and openstacksdk to add support for new floatingip's
attribute. To enable it for floatingip, it should be something like:

::

    openstack floating ip create --distributed

and to disable it:

::

    openstack floating ip create --centralized

and update it:

::

    openstack floating ip update [--distributed | --centralized ]

Testing
-------

* Unit tests.
* Functional test.
* Tempest tests in neutron-tempest-plugin.

Assignee(s)
-----------

* ZhouHeng <zhouhenglc@inspur.com>

References
==========


.. [1] Neutron Drivers meeting, June 24, 2022
   https://meetings.opendev.org/meetings/neutron_drivers/2022/neutron_drivers.2022-06-24-14.00.log.html#l-116