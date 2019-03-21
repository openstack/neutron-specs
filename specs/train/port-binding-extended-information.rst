..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Port binding records, capabilities and events
=============================================

https://bugs.launchpad.net/neutron/+bug/1821058

There are several cases in the Nova/Neutron/os-vif interaction where the
knowledge of the neutron core plugin or ML2 driver would be useful to
facilitate a more robust handling of guest networking. For example,
[3]_.

This spec aims to enhance the port binding API via the introduction of a new
extension to provide the additional information required to enable Nova and
os-vif to make more intelligent use of port binding info.

Please remember os-vif was only intended to remove the VIF plugging process
from Nova. The hypervisor configuration (e.g.: libvirt XML generation) is still
Nova's responsibility. Although os-vif removes the networking back-end logic
related to the plug and unplug process, Nova still needs to handle the
information proposed in this spec.


Problem Description
===================

This spec aims to resolve 3 related problems.

Network connectivity
--------------------

[3]_ blueprint seeks to enable the use of instances with ip_allocation=none.
The port parameter "ip_allocation" was introduced in the [1]_, to "distinguish
the unaddressed port case from the deferred IP allocation case where routed
networks is involved" (extracted from [2]_).

To do this safely and guarantee the guest will have network connectivity, Nova
must ensure that the network backend that bound the port provides L2
connectivity.

For example, in a mixed Calico/SR-IOV deployment, it is valid to use
ip_allocation=none for any port that is bound by the SR-IOV ML2 driver, but not
for the Calico backend. In this case, the only connectivity Calico provides to
a guest is L3, therefore is not possible to spawn a VM without an IP address.

Network isolation
-----------------

In resolving [4]_, a new config option was required in os-vif to allow
enabling VIF isolation for the ML2/ovs backend. As Nova cannot differentiate
between VIF_TYPE=”ovs” that is bound by ML2/odl or ML2/ovs it cannot
automatically instruct os-vif to enable or disable VIF isolation for ML2/ovs
hosts. If the ML2 driver(s) that bound the port are recorded in the port
binding details, Nova and os-vif can use that information to more
intelligently enable backend specific code paths without requiring additional
network backend specific config options.

Bandwidth scheduling
--------------------

The network driver information can be used too for the bandwidth scheduling
featured implemented in Stein, as pointed out in this [6]_. Currently Nova
cannot handle the case where the PF is used by one ML2 driver while the VF is
managed by the SR-IOV ML2 driver.


Proposed Change
===============

To address the 3 problems stated above, the following additions to the
``binding:vif_details`` dictionary in the port object are proposed:

* A new ``connectivity`` field will be introduced with allowed values of "l2",
  "l3" and "legacy". The "legacy" ``connectivity`` value will be the default
  and is a sentinel to indicate the driver does not support this extension yet
  and no connectivity info is available. "l2" drivers provide layer 2
  connectivity, like for example Linux Bridge, Open vSwitch or SR-IOV. "l3"
  drivers provide only layer 3 connectivity, like for example Calico [8]_.

* A new ``bound_drivers`` field will be added that is a dictionary mapping
  binding_level to driver name. The driver name will be the stevedore entry
  point name for the driver which is already used in config files and intended
  to be stable. A POC is available in [7]_.


To enable declaration of the ``connectivity``, a new property will be added to
the ML2 driver base class which will default to "legacy". ML2 drivers that
inherit from this will override the property.

The ``bound_drivers`` field will be used by the OVS os-vif plugin to remove the
need for [5]_. We can enable this option only when
needed.


Examples
--------

ML2/ovs:

.. code-block:: python

    binding_details: {
        ...
        "connectivity": "l2",
        "bound_drivers": {"0": "openvswtich"}
    }


ML2/odl v1:

.. code-block:: python

    binding_details: {
        ...
        "connectivity": "legacy",
        "bound_drivers": {"0": "opendaylight"}
     }


ML2/odl v2:

.. code-block:: python

    binding_details: {
        ...
        "connectivity": "l2",
        "bound_drivers": {"0": "opendaylight_v2"}
    }


ML2/calico

.. code-block:: python

    binding_details: {
        ...
        "connectivity": "l3",
        "bound_drivers": {"0": "calico"}
    }


References
==========

.. [1] `BP Allow vm to boot without l3 address`:
       https://blueprints.launchpad.net/neutron/+spec/vm-without-l3-address

.. [2] `commit 361455`:
       https://review.openstack.org/#/c/361455/

.. [3] `Boot a VM with an unaddressed port`:
       https://blueprints.launchpad.net/nova/+spec/boot-vm-with-unaddressed-port

.. [4] `bug 1734320`:
       https://bugs.launchpad.net/neutron/+bug/1734320

.. [5] `isolate_vif config option`:
       https://github.com/openstack/os-vif/blob/stable/stein/vif_plug_ovs/ovs.py#L146-L159

.. [6] `review 1`:
       https://review.opendev.org/#/c/623543/40/nova/compute/manager.py@2161

.. [7] `review 2`:
       https://review.openstack.org/#/c/635083

.. [8] `Calico project`:
       https://docs.openstack.org/networking-calico/latest/
