..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================================
Big Switch - Support for External Attachment Point Extension
============================================================

https://blueprints.launchpad.net/neutron/+spec/bsn-external-attachment

Add support for the external attachment point extension to the Big Switch
plugin so it can provision arbitrary ports in the fabric to be members of
Neutron networks.


Problem description
===================

Neutron lacked a way to express attachments of physical ports into Neutron
networks. For this the external attachment specification was approved.[1]
However, that spec only covers ML2 deployments so it will need to be included
in the Big Switch plugin for users of the plugin to gain this feature.


Proposed change
===============

Include the extension mixin in the Big Switch plugin and add the appropriate
REST calls to the backend controller to provide network access to these
external attachment points.

Alternatives
------------

N/A

Data model impact
-----------------

The Big Switch plugin will need to be included in the DB migration for the
external attachment tables.

REST API impact
---------------

The Big Switch plugin will support the external attachment REST endpoints.

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

* Include the mixin and create the methods to bind the vlan_switch type via
  a REST call to the backend controller
* Add unit tests

Dependencies
============

Implementation of the extension. [1]


Testing
=======

At first this will be covered by regular unit tests and the integration test
for the OVS gateway mode. An additional 3rd party CI test will be setup
to exercise the custom provisioning code.


Documentation Impact
====================

Add mention of support of this feature for the Big Switch plugin.

References
==========

1. https://github.com/openstack/neutron-specs/blob/master/specs/juno/neutron-external-attachment-points.rst
