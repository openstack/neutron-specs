..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Port extension to create hardware offloaded ports
=================================================

Launchpad bug: https://bugs.launchpad.net/neutron/+bug/2013228

The aim of the RFE is to create a new port extension to allow to create ports
with hardware offloaded capabilities.


Problem Description
===================

The RFE [1]_ introduced the capability of creating a port that could use the
hardware offload support implemented in Open vSwitch [2]_. This feature was
implemented in the following set of patches (initially for ML2/OVS): [3]_,
[4]_ and [5]_.

In order to create a port with hardware offload capabilities, it is needed to
define the VNIC type and set a specific value in the port binding profile::

    openstack port create --vnic-type direct \
        --binding-profile '{"capabilities": ["switchdev"]}' port_hwol


This method to create a port has several drawbacks:

* The port binding profile field is an admin only parameter by default.
  The values contained in this field can have references to PCI addresses of
  the host and should not be modified by a non-admin user.
* Actually this parameter should not be written by Neutron, only by Nova when
  the port is bound.


Proposed Change
===============

This RFE proposes to create a new port extension, called
"port-hardware-offload", that will replace the need of manually defining the
port binding profile information when creating the port; please note that
this is referring only to the port creation process. This API extension
consists of a string parameter that will be "None" by default. This string
will be the hardware offload type; OpenStack Neutron and Nova are currently
supporting only "switchdev" [6]_. The strings allowed by the API will be
limited to a set of defined constants.

This new string parameter will be passed to Nova along with the port
information dictionary. Nova will use this parameter instead of the port
binding profile information to command ``os-vif`` to create the
corresponding layer 1 port (a devlink port [3]_ [7]_).

Because of this Nova-Neutron communication, this RFE is going to be implemented
in two steps:

* The first one will implement only the Neutron and OSC (plus the OpenstackSDK
  bits) code. The port dictionary passed to Nova will contain both the new
  parameter added in this RFE and the port binding profile, that will be added
  internally by Neutron when the new paramter is not "None".

* The second step involves the Nova implementation. This change implies that
  Nova can read the port new parameter and create the corresponding port. The
  port binding profile information passed by Neutron will be irrelevant.

.. NOTE::

    The scope of this RFE, as discussed during the presentation of this new
    feature in the Neutron drivers meeting, involves only the first step. The
    Nova code change will be covered in other RFE and spec.


Client Impact
-------------

The OSC and OpenstackSDK projects will be updated in order to be able to create
a port using this new extension. Port creation example::

    openstack port create --vnic-type direct \
        --hardware-offload-type switchdev port_hwol


The new string field will be visible when showing the port resource.

This spec is not covering the Nova migration from reading the port binding
profile to using the new parameter provider in the port definition. As
commented in the first section, Neutron will still populate the port binding
profile as long as the "hardware-offload-type" is provided in the port
creation command. In order to stop users to create a port with the old
method, the OSC will check if the "capabilities" key is provided inside the
port binding profile and a warning will be printed in the command line,
encouraging the user to create the port using the "hardware-offload-type"
parameter.


REST API Impact
---------------

Proposed attribute::

    RESOURCE_ATTRIBUTE_MAP = {
        port.COLLECTION_NAME: {
            "hardware_offload_type": {
                'allow_post': True,
                'allow_put': False,
                'convert_to': converters.convert_to_string,
                'default': None,
                'is_visible': True,
                'is_filter': True,
                'validate': {
                    'type:values': constants.VALID_HWOL_TYPES}
            }
        }
    }


Sample REST calls::

    POST /v2.0/ports
    {
        "port": {
           ...
           "hardware_offload_type": "switchdev"
    }

    Response:
    {
        "port": {
            "id": "<id>",
            ...
            "hardware_offload_type": "switchdev"
        }
    }


Data Model Impact
-----------------

This RFE proposes to create a child table to ``ports``, called
``port_hardware_offload``.

.. table:: **Port hardware offload**

    ===================== ======== ==== =====================================
    Attribute             Type     CRUD Description
    ===================== ======== ==== =====================================
    port_id               uuid-str CR   Unique identifier for the port object
    hardware_offload_type str      CR   String to indicate the hardware
                                        offload type
    ===================== ======== ==== =====================================


Security Impact
---------------

By default, this new field will be writable by the admin only. However, the
admin can consider changing the rule owner and allow any project user to create
a port defining this parameter::

    policy.DocumentedRuleDefault(
        name='create_port:hardware_offload_type',
        check_str=base.ADMIN,
        scope_types=['project'],
        operations=ACTION_POST
    )


This rule change was completely discouraged for the rule
'create_port:binding:profile' for the reasons provided in the problem
description.


Performance Impact
------------------

The port resource view will now require a new "JOIN" operation between the
``ports`` table and the ``ports_hardware_offload`` table when building the
port OVO. However there will be a 1:1 relationship between both tables and the
data retrieved from the child table is minimal (two columns).


Other Impact
------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
  Rodolfo Alonso Hernandez <ralonsoh@redhat.com> (IRC: ralonsoh)

Work Items
----------

* API implementation (neutron-lib and Neutron).
* Database migration (Neutron)
* CLI implementation (OpenstackSDK and OSC)
* Documentation.
* Tests and CI related changes.


Testing
=======

* Unit/functional tests.
* Fullstack API tests.


Documentation Impact
====================

User Documentation
------------------

Document the new way to create hardware offload ports and deprecate the older
method.


References
==========

.. [1] [RFE] SR-IOV accelerated OVS integration
       https://bugs.launchpad.net/neutron/+bug/1627987
.. [2] https://mail.openvswitch.org/pipermail/ovs-dev/2017-April/330606.html
.. [3] https://review.opendev.org/c/openstack/os-vif/+/460278
.. [4] https://review.opendev.org/c/openstack/neutron/+/275616
.. [5] https://review.opendev.org/c/openstack/neutron/+/499203
.. [6] https://docs.kernel.org/networking/switchdev.html
.. [7] https://www.kernel.org/doc/html/next/networking/devlink/devlink-port.html
