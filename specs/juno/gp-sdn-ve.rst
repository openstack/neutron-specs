..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Group Based Policy support with IBM SDN-VE Controller Drivers
=============================================================

Launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/group-based-policy-with-sdn-ve

This blueprint proposes the support of the Group Based Policy
extension with the IBM SDN-VE controller.


Problem description
===================

The Group Based Policy (GBP) blueprint has proposed new application
centric APIs for Neutron. In order to utilize these new APIs with
SDN-VE, it is necessary to provide the driver for implementing and
proxying the new GBP APIs to SDN-VE.


Proposed change
===============

We propose the addition of a new driver in the context of the GBP
service plugin to support the implementation the GPB API by using
SDN-VE.

Alternatives
------------

One possible option is not utilizing the GBP service plugin framework
and either provide a new service plugin to be use by either the
monolithic or the ML2 based SDN-VE driver. Another alternative is to
add the support for the GBP extension in the monolithic SDN-VE.
Considering that reusing the GBP service plugin will reduce code
replication and significantly reduce the cost of maintaining the
code, the alternative options do not seem cost effective.


Data model impact
-----------------

None, as this change is simply adding support for new API extensions.

REST API impact
---------------

The new GBP API extensions are supported.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

Users deploying Juno with the Group Based Policy extensions will be able to
utilize these new APIs in conjunction with SDN-VE.

The proposed addition does not have an impact on python-neutronclient.

Performance Impact
------------------

None.

Other deployer impact
---------------------

None.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary Assignee:
  Mamta Prabhu <mamtaprabhu>
Other contributors:
  Mohammad Banikazemi (banix) <mb-s>
  Ryan Moats <regXboi>

Work Items
----------

 SDN-VE driver for the GBP Service Plugin

Dependencies
============

The following patch sets under review implement the data model and the
service plugin framework. The proposed driver will fit in this frame
work.

* GBP Model: Group Policy DB-1: EP, EPG, L2 Policy, L3 Policy:
  https://review.openstack.org/#/c/96050/

* GBP Service Plugin: Group Policy Plugin-1: EP, EPG, L2 Policy, L3
  Policy: https://review.openstack.org/#/c/96393/

Testing
=======

The SDN-VE 3rd part CI system will be utilized to cover the testing of
the additional extension.

Documentation Impact
====================

The documentation should reflect the spport of GBP extension by the
IBM SDN-VE controller.

References
==========

* Group Policy Wiki:
  https://wiki.openstack.org/wiki/Neutron/GroupPolicy

* Group-based Policy Abstractions for Neutron:
  https://review.openstack.org/#/c/89469/
