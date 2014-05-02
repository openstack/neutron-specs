
=============================================
Extraroute Extension support for Nuage Plugin
=============================================

https://blueprints.launchpad.net/neutron/+spec/extraroute-ext-support-for-nuage-plugin

Adding extraroute extension support to existing nuage networks' Plugin


Problem description
===================
Current Nuage Plugin does not support Neutron's extra-route extension.
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
There are couple of extra tables which will be needed to support this.
One is routerroutes which already exist in neutron. No schema changes required
for that table.
Second table is Nuage plugin specific. This will be added as part of the new
code.

It could look like::

    class RouterRoutesMapping(model_base.BASEV2, models_v2.Route):
        __tablename__ = 'routerroutes_mapping'
        router_id = Column(String(36), ForeignKey('routers.id', ondelete="CASCADE"),
                    primary_key=True)
        nuage_route_id = Column(String(36))

migration script will need to be generated.

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
Nuage's VSP supports adding static-route to L3 Domain which fits nicely
with extraroute extension supported by openstack's neutron.

Following is a mapping of the neutron's work-flow for nuage plugin:

Assume router1 exist.
STEP1:
neutron router-update router1 --routes type=dict list=true \
destination=15.0.0.0/24,nexthop=10.10.0.5
This will create a staticroute entry in l3-domain router1 in VSP

STEP2:
neutron router-update router1 --routes type=dict list=true \
destination=15.0.0.0/24,nexthop=10.10.0.5 destination=25.0.0.0/24,nexthop=10.10.0.6
This will add one more staticroute to the same.

STEP3:
neutron router-update router1 --routes type=dict list=true \
destination=15.0.0.0/24,nexthop=10.10.0.5
This will delete staticroute of STEP2.

Basically everytime you specify routes option it treats it as a new set of value.
So sticking to that same behavior.

STEP4:
neutron router-update router1 --routes action=clear
Cleans up all the staticroute.


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
Unit Test coverage for extra-route extension within Nuage unit test
Nuage CI will be modified to start supporting this extension tests


Documentation Impact
====================
None

References
==========
None
