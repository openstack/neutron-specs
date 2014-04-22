
=============================================
FloatingIP Extension support for Nuage Plugin
=============================================

https://blueprints.launchpad.net/neutron/+spec/floatingip-ext-for-nuage-plugin

Adding floatingip extension support to existing nuage networks' Plugin


Problem description
===================
Current Nuage Plugin does not support Neutron's floatingip extension.
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
Existing floatingip tables in neutron will be supported.
On top of that, there will be 2 nuage specific tables which will be added:
Schema could look like::

    class FloatingIPPoolMapping(model_base.BASEV2):
        __tablename__ = "floatingip_pool_mapping"
        fip_pool_id = Column(String(36), primary_key=True)
        net_id = Column(String(36), ForeignKey('networks.id', ondelete="CASCADE"))
        router_id = Column(String(36), ForeignKey('routers.id', ondelete="CASCADE"))

    class FloatingIPMapping(model_base.BASEV2):
        __tablename__ = 'floatingip_mapping'
        fip_id = Column(String(36), ForeignKey('floatingips.id',ondelete="CASCADE"),
                        primary_key=True)
        router_id = Column(String(36))
        nuage_fip_id = Column(String(36))


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
Its a straightforward floatingip extension support where
resource from neutron will be mapped into VSP.
When user creates external network and gives it a subnet,
floatingip-pool will be created.
When floatingip is created, its created in VSP as well.
Associate, disassociate of floatingip with port should work
in the same way as well.

CRUD operation on floatinip will be supported in normal fashion.

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
Unit Test coverage for floating-ip extension within Nuage unit test
Nuage CI will be modified to start supporting this extension tests


Documentation Impact
====================
None

References
==========
None
