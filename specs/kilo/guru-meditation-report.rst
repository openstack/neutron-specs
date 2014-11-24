..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Add Guru Meditation Report Functionality to Neutron
===================================================

Guru Meditation Report functionality has been implemented in Oslo. The trigger
code now needs to be added to the Neutron executables so that they print a GMR
on SIGUSR1 (see https://wiki.openstack.org/wiki/GuruMeditationReport)

Problem Description
===================

GMR can aid in debugging neutron by providing lots of detail about a running
process, including configuration, native thread stack trace and greenlet stack
traces.

GMR is intended to be triggered by sending USR1 signal to a neutron process, If
the process is a daemon, the report will be placed inside logdir. Otherwise,
the report will output to the console.

Proposed Change
===============

GMR has already been implemented in olso-incubator, pick GMR from olso and add
trigger code.

GMR will be supported by all neutron daemons, including:
* neutron-cisco-cfg-agent
* neutron-db-manage
* neutron-dhcp-agent
* neutron-hyperv-agent
* neutron-ibm-agent
* neutron-l3-agent
* neutron-lbaas-agent
* neutron-linuxbridge-agent
* neutron-metadata-agent
* neutron-mlnx-agent
* neutron-nec-agent
* neutron-ns-metadata-proxy
* neutron-nsx-manage
* neutron-nvsd-agent
* neutron-openvswitch-agent
* neutron-restproxy-agent
* neutron-ryu-agent
* neutron-server
* neutron-vpn-agent
* neutron-metering-agent
* neutron-ofagent-agent
* neutron-sriov-nic-agent

Data Model Impact
-----------------

None.

REST API Impact
---------------

None.

Security Impact
---------------

The configuration can be viewed by the user who can read logdir. But secrets
(including passwords, tokens, database connections) will be masked out.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

IPv6 Impact
-----------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

None.

Community Impact
----------------

None.

Alternatives
------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Zang MingJie <zealot0630@gmail.com>

Work Items
----------

* Sync GRM from olso
* Add trigger code

Dependencies
============

None.

Testing
=======

Tempest Tests
-------------

None.

Functional Tests
----------------

* Test GMR trigger. When receive a USR1 signal, the corresponding GMR function
  muse be called. The generating progress should be well tested in GMR tests.

API Tests
---------

None.

Documentation Impact
====================

Documentation to describe how to use the feature. Should be almost the same as
in nova. [ref 2: GMR in Nova]

User Documentation
------------------

Describe how to generate a GMR, and locate the generated GMR.

Developer Documentation
-----------------------

Describe how to write GMR trigger and extend the GMR.

References
==========

* Guru Meditation Report: https://wiki.openstack.org/wiki/GuruMeditationReport
* GMR in Nova: http://docs.openstack.org/developer/nova/devref/gmr.html
