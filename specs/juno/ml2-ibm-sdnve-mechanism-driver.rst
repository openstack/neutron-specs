==============================================
ML2 Mechanism Driver for IBM SDN-VE Controller
==============================================

Launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/ml2-ibm-sdnve-mechanism-driver
This blueprint is for adding support for IBM SDN-VE controller through
a new mechanism driver.

Problem description
===================

SDN-VE is a controller that provides network virtualization and
support for software defined networking. In order to utilize SDN-VE
with OpenStack, we need to provide a plugin or mechanism driver for
this controller.

Proposed change
===============

We propose the addition of a new ML2 mechanism driver for supporting
SDN-VE controller. The proposed mechanism driver will mainly act as a
pass through driver responsible for passing incoming Neutron requests
to the controller through the SDN-VE REST API (after preprocessing the
requests as needed and also processing received responses from the
controller).  We will use an agent (the SDN-VE agent) for configuring
the virtual switches on compute nodes. We will utilize the existing
RPC infrastructure for communication between the mechanism driver and
the agent. The proposed mechanism driver performs port binding and
sets the vif_type for a given port to either OVS or Bridge depending
on the tenant type.

The tenant type is determined from extra information that is possibly
associated with a tenant (project) when it is created through
Keystone. The default tenant type (a configuration parameter) is used
when this information is not provided. The tenant type is determined
when a given tenant creates a network for the first time. This
information is then used for port binding when ports get created on
the given network.

In developing the mechanism driver for SDN-VE, we have recognized the
logic shared among SDN-VE driver and other drivers for SDN
controllers, namely the ODL driver. We will identify the code that can
be shared by such drivers and develop a framework for reusing the code
to the extent possible. If this attempt is successful, we will pursue
changes to the ODL driver through a separate spec/blueprint (and set
of work items).


Alternatives
------------

The alternative would be to continue using the monolithic driver for
SDN-VE. In order to simplify the controller and in order to take
advantage of the support for various extensions already available in
ML2 and to avoid replicating code as much as possible and in order to
simplify the maintenance of our code we have decided to make the move
to ML2 and develop a mechanism driver for it. Upon successful
implementation and integration of the new mechanism driver we will
plan for discontinuing the monolithic plugin.

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

The SDN-VE mechanism driver specific configuration parameters that are
to be provided by deployers will be specified in the updated
documentation. These parameters include the address and credentials
required for accessing the SDN-VE controller.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Mamta Prabhu <mamprabhu>
Other contributors:
  Mohammad Banikazemi (banix) <mb-s>

Work Items
----------

1. Developing a library for communication with SDN-VE
2. Developing the mechanism driver
3. Possible addition to the current SDN-VE agent
4. We will work towards deprecating the monolithic SDN-VE plugin after
   this mechanism driver is successfully merged.

Dependencies
============

none

Testing
=======

Since access to the SDN-VE controller is required for testing the
proposed cahnges, the 3rd party testing is essential. We will use the
current CI being used for the monolithic SDN-VE plugin while we create
a new and improved zuul based CI system.

Documentation Impact
====================

Documentation needs to be updated to reflect the addition of a new
mechanism driver with new configuration parameters.


References
==========

None
