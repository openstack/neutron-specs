
=================================================
SecurityGroup  Extension support for Nuage Plugin
=================================================

https://blueprints.launchpad.net/neutron/+spec/securitygroup-ext-for-nuage-plugin

Adding securitygroup extension support to existing nuage networks' Plugin


Problem description
===================
Current Nuage Plugin does not support Neutron's securitygroup extension.
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
Existing securitygroup tables in neutron will be supported.

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
VSP's securitygroup equivalent object's scope is either per router or per subnet.
Where Neutron's is per tenant. Because of this, the mapping between
neutron and VSP resource always happens at the port create or update time; such
that port's router/subnet is known and thus sg attachment point in VSP is known.
Following workflow can be imagined:
1) neutron security-group-create sg1
No-op from VSP point of view
2) neutron security-group-rule-create --direction ingress --protocol tcp --port_range_min 80 --port_range_max 80 <sg-id>
No-op from VSP point of view
3a) neutron port-create 9d0b9f4a-1a72-4c17-a538-06ee7501d185 --name sub1 --security-group 8eb7ee8e-6d15-4a0d-b13a-0affeba438ae
3b) neutron port-update 71083f7d-1450-4bee-9c40-728b7ffd2876 --security-group c6c08246-bad7-4d82-a0ad-4a42327c9516
If this is the first port getting attached to that security-group,
this is where corresponding vport-tag (for sg) and rules (for sg-rules) are created on VSP.
Subsequent port-create/update for this sg will simply increment counter and add value to vport to vporttag
mapping.

Similarly, when the last port attached to this group is deleted, the vport-tag(sg) and the rules(vptag rules)
will be deleted.

CRUD operation on securitygroup will be supported in normal fashion.

Assignee(s)
-----------
Ronak Shah


Primary assignee:
  ronak-malav-shah

Other contributors:
  divya.hc

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
Unit Test coverage for security-group extension within Nuage unit test
Nuage CI will be modified to start supporting this extension tests


Documentation Impact
====================
None

References
==========
None
