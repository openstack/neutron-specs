
==============================================
Providernet Extension support for Nuage Plugin
==============================================

https://blueprints.launchpad.net/neutron/+spec/providernet-ext-support-for-nuage-plugin

Adding provider-network extension support to existing nuage networks' Plugin


Problem description
===================
Current Nuage Plugin does not support Neutron's providernet extension.
Nuage's VSP supports this feature and the support for extension needs
to be added in the plugin code.

Proposed change
===============
Adding extension support code in Nuage plugin.


Alternatives
------------
None

Data model impact
-----------------
None

REST API impact
---------------
None

Security impact
---------------
None

Notifications impact
--------------------
None

Other end user impact
---------------------
None

Performance Impact
------------------
None

Other deployer impact
---------------------
None

Developer impact
----------------
None

Implementation
==============
The provider extended attributes for networks enable administrative users to specify how
network objects map to the underlying networking infrastructure. This fits well with
Nuage's ability to map the overlay with the underlay provided option. There will not be a
significant change in the plugin behavior. When the network is created with provider
options enabled, segments information (id, physical_network, vlan_id) is passed to the
Nuage's backend and it will be acted upon. Backend will either accept it or will fail
the request and the response will be relayed back to the user. In case of a failure,
resource operation in neutron will also fail. In Juno, we want to keep the implementation
most basic.

Assignee(s)
-----------
Ronak Shah


Primary assignee:
  ronak-malav-shah

Other contributors:

Work Items
----------
Extension code in Nuage plugin
Nuage Unit tests addition
Nuage CI coverage addition


Dependencies
============
None

Testing
=======
Unit Test coverage for providernet extension within Nuage unit test
Nuage CI will be modified to start supporting this extension tests as and when
corresponding tests are added in tempest


Documentation Impact
====================
None

References
==========
None
