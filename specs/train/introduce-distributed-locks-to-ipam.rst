..
     This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Introduce distributed locks to ipam module
==========================================

https://blueprints.launchpad.net/neutron/+spec/introduce-distributed-locks-to-ipam
RFE:https://bugs.launchpad.net/neutron/+bug/1836834

Introduce the OpenStack tooz distributed lock to the ipam module. In the scenario
of large-scale port creation, avoid ip allocation exceeding the maximum retry
limit failure and improve ip allocation efficiency.


Problem Description
===================
#. Current port creation has a probability of failure.

   When the virtual machines are created in batches, nova will call the neutron
   API to create the ports concurrently. Port creation will fail if there is an
   ip allocation conflict, see [1]_.

   And bulk create port has the similiar problem, when multiple "create_port_bulk"
   APIs are called simultaneously on the same subnet, although this scenario is
   used less frequently.

#. When creating ports concurrently, the utilization ratio of neutron
   server CPU increases and the allocation efficiency decreases.

   When an ip allocation conflict fails to submit a database, a ``DB ERROR``
   exception is thrown. ``Create_port`` will catch the above exception and rest
   after ``retry_interval=0.1`` and re-call ``create_port`` until it exceeds
   ``max_retries=10``. When it exceeds ``max_retries=10`` times, "Create_port"
   will fail. When concurrency is large and conflict intensifies, repeated call
   to create_port increases CPU's burden and reduces allocation efficiency.
   Adjusting ``retry_interval`` and ``max_retries`` can only reduce the
   probability of problems, but can not solve them thoroughly.

Proposed Change
===============
This solution implements a new ipam driver by introducing a distributed lock
to completely solve the problem of ip address allocation conflict leading
to failure.

For distributed locks we will use OpenStack tooz [2]_ , which supports many
backend drivers, such as Zookeeper, Memcached, Redis, Mysql, etc., and it is
an OpenStack native project. We will support the configuration of tooz backend
drivers in neutron.conf, such as adding [tooz] configuration items.

The new IPAM allocate ip seqdiag.
   .. seqdiag::

    seqdiag {
        "Neutron Plugin"  ->  NeutronDbPluginV2 [label = "create_port"];
        NeutronDbPluginV2 -> "IPAM Driver" [label = "get_subnet"];
        NeutronDbPluginV2 <- "IPAM Driver" [label = "IPAMSubnet"];
        NeutronDbPluginV2 -> IPAMSubnet [label = "allocate_ip"];
        IPAMSubnet -> "Pluggable IPAM" [label = "Allocate IP"];
        "Pluggable IPAM" -> "Pluggable IPAM" [label = "lock ip by ip+subnet_id"];
        IPAMSubnet <- "Pluggable IPAM" [label = "IP"];
        NeutronDbPluginV2 <- IPAMSubnet [label = "IP"];
        NeutronDbPluginV2 -> "Neutron DB" [label = "port, IP data"];
        NeutronDbPluginV2 <- "Neutron DB";
        "Neutron Plugin" <- NeutronDbPluginV2 [label = "port, IP data"];
        "Neutron Plugin" -> "Neutron Plugin" [label = "release lock"];
    }

Alternatives
------------
* We can modify the current ipam driver to introduce distributed locks to solve
  the above mentioned problems when creating a new ipam driver is not feasible.

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
* Operator can configure the backend driver for tooz using the [tooz] configuration
  block in neutron.conf.

  .. code-block::

    [tooz]
    # Tooz backend connection string.
    backend_url = file://$state_path

    # Number of seconds between heartbeats for distributed coordination.
    heartbeat = 1.0

    # Number of seconds to wait after failed reconnection to Tooz backend.
    initial_reconnect_backoff = 0.1

    # Maximum number of seconds between sequential reconnection retries to Tooz backend.
    max_reconnect_backoff = 60.0

* Operator can switch to our new ipam driver by setting "ipam_driver" in neutron.conf.

  .. code-block::

    [default]
    # Neutron IPAM (IP address management) driver to use. By default, the reference
    # implementation of the Neutron IPAM driver is used. (string value)
    ipam_driver = ipam_with_dlm

Performance Impact
------------------
When using rally to test concurrently to create vms or ports or the similar scenes.

The good aspects:

* Solve the failure of creating vms or ports due to ip allocation conflicts, and
  improve the success rate.
* Reduced average time to create vms or ports.
* Create vms or ports with a smoother distribution of time.

Minor impact:

* The minimum time to create vms or ports has increased slightly, and creating a
  port time in a non-concurrent scenario will also increase slightly.

Other deployer impact
---------------------
None

Developer impact
----------------
None

Implementation
==============

Assignee(s)
-----------
Primary assignee:
  qinhaizhong

Other contributors:
  zhouhenglc

Work Items
----------
* Create a new ipam driver.
* Support for parsing [tooz] backend drivers, encapsulating distributed lock modules,
  and implementing distributed lock initialization, locking, unlocking, etc.
* Make "create_port" to support the new ipam driver.
* Make "bulk_create_port" to support the new ipam driver.
* Documentation work.

Dependencies
============
None

Testing
=======
Unit tests, functional tests.

Documentation Impact
====================
None

References
==========

.. [1] https://bugs.launchpad.net/neutron/+bug/1777968
.. [2] https://launchpad.net/python-tooz

