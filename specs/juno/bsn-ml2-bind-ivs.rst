..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Big Switch - Have ML2 driver bind IVS VIF type
==============================================

https://blueprints.launchpad.net/neutron/+spec/bsn-ml2-bind-ivs

The Big Switch ML2 driver needs to be responsible for binding the IVS VIF type
since there is no binding agent running for this virtual switch.


Problem description
===================

Currently, the Big Switch ML2 driver is only responsible for configuring VLANs
in the fabric. It doesn't do anything with the vSwitch so it leaves the
responsibility of binding the ports to the OpenVSwitch driver. However, it
needs to support IVS switches as well, which will not have a binding driver
since they are directly controlled by the backend controller. Therefore, it
will need to be responisble for responding to the bind_port calls for hosts
with the IVS switches and mark them as bound.


Proposed change
===============

At the port binding method to the Big Switch ML2 driver and have it respond
to port binding calls for IVS ports.

Alternatives
------------


Data model impact
-----------------


REST API impact
---------------


Security impact
---------------


Notifications impact
--------------------

Other end user impact
---------------------


Performance Impact
------------------


Other deployer impact
---------------------
Deployers will be able to deploy IVS vswitches with the Big Switch ML2 plugin.

Developer impact
----------------



Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kevinbenton

Work Items
----------

* Add the port binding methods to the ML2 driver
* Add unit tests


Dependencies
============

N/A

Testing
=======

In addition to the unit tests, a new 3rd party CI test case will be added
for IVS deployed under ML2.

Documentation Impact
====================

Mention that IVS can be used in conjuction with Big Switch Ml2 deployments.

References
==========

N/A
