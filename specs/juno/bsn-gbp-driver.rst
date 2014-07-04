..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Big Switch Group Based Policy Driver
====================================


https://blueprints.launchpad.net/neutron/+spec/bsn-gbp-driver

The version of the Big Switch controller currently under development targeted
for Juno will have a native group policy API compatible with the OpenStack
group policy constructs.


Problem description
===================

The group policy to legacy network object mappings make it difficult for the
backend controller to understand the intent of the original policy since it
only receives the resulting port/subnet/network creation calls. This prevents
it from making optimizations about grouping and enforcement that perform better
on the fabric than the standard subnet/network groups.


Proposed change
===============

Create a new group policy driver that will communicate directly with
a group policy API on the Big Switch controllers rather than rely on
the mapping to standard Neutron objects. This allows the backend to make
decisions about how to implement the grouping based on fabric optimizations.

Alternatives
------------

Rely on the default GP mapping driver that converts endpoint groups into
standard neutron objects. This does not let the controller perform grouping
optimizations that work better for the network hardware.


Data model impact
-----------------

N/A

REST API impact
---------------

This will conform to the standard group policy REST API.

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

This will add a new driver that a deployer can configure to use native group policy
in Big Switch deployments.


Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kevinbenton

Work Items
----------

* Write a driver that maps group policy calls in neutron to the corresponding
  REST calls to the backend controller.
* Add unit tests
* Add 3rd party CI


Dependencies
============

The group policy framework being implemented this cycle.[1]


Testing
=======

This will be covered by unit testing and then 3rd party CI testing once the
backend implementation is complete.


Documentation Impact
====================

Show how to load the Big Switch group policy driver.
The backend configuration should be consistent with the current Big Switch
plugin so the extra configuration should be minimal.


References
==========

1. https://wiki.openstack.org/wiki/Neutron/GroupPolicy
