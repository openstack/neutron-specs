..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Big Switch - Support for Provider Networks
==========================================

https://blueprints.launchpad.net/neutron/+spec/bsn-provider-net

Add support for provider networks to the Big Switch plugin to make connecting
vswitches outside of the control of the fabric possible.


Problem description
===================

When the backend controller controls all of the physical and virtual switches,
the chosen segmentation ID is irrelevant to Neutron. Since this was the only
supported model for the Big Switch plugin, it didn't need the provider net
extension to specify segmentation IDs for each network.

However, the plugin now needs to support heterogeneous environments containing
vswitches under the control of the controller and standalone vswitches
controlled by a neutron agent. To support these latter switches, a segmentation
ID is required so the agent understands how to configure the VLAN translation
to the physical network.


Proposed change
===============

Implement the provider network extention in the plugin to populate the
segmentation ID for vlan networks.


Alternatives
------------

N/A. This VLAN information is not available to neutron unless it selects it on
its own rather then leaving it up to the fabric.


Data model impact
-----------------

N/A


REST API impact
---------------

Port and network responses for admins will contain segmentation information.

Security impact
---------------

N/A

Notifications impact
--------------------

N/A

Other end user impact
---------------------

N/A

Performance Impact
------------------

N/A

Other deployer impact
---------------------

N/A

Developer impact
----------------

N/A

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kevinbenton


Work Items
----------

* Implement extension in Big Switch plugin
* Add unit tests to ensure segmentation ID is present


Dependencies
============

The bsn-ovs-plugin-agent spec will depend on this support being added.[1]

Testing
=======

Unit tests will cover the addition of this extension. After the dependent
blueprint is implemented, the 3rd party CI system will exercise this code.

Documentation Impact
====================

N/A

References
==========

1. https://blueprints.launchpad.net/neutron/+spec/bsn-ovs-plugin-agent

