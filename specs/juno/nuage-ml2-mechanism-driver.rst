
=============================================
Nuage Networks' ML2 Mechanism Driver
=============================================

https://blueprints.launchpad.net/neutron/+spec/ml2-mech-driver-nuage

Adding mechanism driver in ML2 plugin for Nuage Networks


Problem description
===================
Nuage's VSP fits well via its northbound API exposure to both ML2 based mechanism
driver as well as via monolithic plugin. As nuage strive to add more features and
functionality to its monolithic plugin, it is important from other integration
point of view to also support ml2 mechanism driver framework. This is the effort
to start achieving that goal.

Proposed change
===============
In Juno release, mechanism driver will support basic L2 functionality as a
stepping stone to enhance it in later release.
It will implement CRUD APIs for network, subnets and ports. Idea is to
reuse as much of the code base from the current monolithic plugin itself and
for that reason driver class will inherit nuage plugin class as one of the
parent.

Alternatives
------------
Monolithic Nuage Plugin

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
In order for mechanism driver to talk to Nuage's VSD, it will
require certain configuration to be read from the file. This is
similar approach to nuage's monolithic plugin. Along with parameters
mentioned at etc/neutron/plugins/nuage/nuage_plugin.ini, user will
have to pass reference to precreated net-partition as well. Once
https://blueprints.launchpad.net/neutron/+spec/neutron-ml2-mechanismdriver-extensions
is implemented (if at all), will add support for it in next release.


Assignee(s)
-----------
Ronak Shah


Primary assignee:
  ronak-malav-shah

Other contributors:

Work Items
----------
ML2 mechanism driver code
Unit tests
Nuage CI infrastructure with ML2 driver (similar to the one with monolithic plugin)

Dependencies
============
None

Testing
=======
Unit Test coverage
Support for this driver in Nuage CI

Documentation Impact
====================
None

References
==========
None
