..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
DHCP Service LoadBalancing Scheduler
====================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/dhcpservice-loadbalancing

Problem Description
===================
With the current Agent Management and Scheduler extensions we are not able to
effectively distribute the DHCP namespaces when there are multiple Network
Nodes available. Existing scheduler(Chance Scheduler) does not load balance
the DHCP namespace properly across multiple Network Nodes based on the network
load of DHCP agents.

Chance scheduler schedules DHCP namespaces unevenly, which is not suitable when
large number of namespaces are created.
Chance scheduler schedules around 90% of DHCP namespaces on a single Network
Node and remaining 10% of namespaces are distributed across remaining Network
Nodes.

This blueprint attempts to address this issue by proposing a new DHCP agent
scheduler which will equally distribute the DHCP namespaces across multiple
Network Nodes based on DHCP namespace count of the DHCP agent.

Proposed Change
===============
In the Neutron server, we have written a DHCP agent scheduler which will
keep track of the DHCP services running on the Network Nodes based on the
"network_scheduler_driver" configuration of 'neutron.conf'.

'LeastNetworksScheduler' type of DHCP scheduler will be triggered only if the
"network_scheduler_driver" parameter is set to LeastNetworksScheduler.
The default value for this flag is 'ChanceScheduler'.

When the new network is created the DHCP agent scheduler will fetch the dhcp
agents with minimum number of networks hosted on it. And schedules the newly
created DHCP namespace service on those minimally loaded DHCP agents.

Here the dhcp agent load is decided based on the number of DHCP namespaces
which are already created on the dhcp agent. 'LeastNetworksScheduler' will
return as many number of minimally loaded DHCP agents as mentioned in
'dhcp_agents_per_network' configuration parameter.

In this implementation the DHCP namespace count is calculated based on report
status messages received from DHCP agents.


Data Model Impact
-----------------
A new column named "load" will be added in the "Agents" table. This column
contains the Agent load based on the DHCP namespaces count which are hosted
on that particular agent. The table will be sorted based on the load and least
loaded top n Agents will be supplied for scheduler to host the DHCP namespaces.

REST API Impact
---------------
None

Security Impact
---------------
None

Notifications Impact
--------------------
None

Other End User Impact
---------------------
None

Performance Impact
------------------
None

IPv6 Impact
-----------
None

Other Deployer Impact
---------------------
This feature can be enabled by setting the flag network_scheduler_driver =
LeastNetworksScheduler which is configurable from "neutron.conf". If the flag
is set to ChanceScheduler, none of this code will be executed.

Developer Impact
----------------
None

Community Impact
----------------
None

Alternatives
------------
None

Implementation
==============

Assignee(s)
-----------
Primary assignee:
  praveen-sm-kumar

Other contributors:
  shiva-m

Work Items
----------

Highlevel tasks include:
   - Refactor the code of ChanceScheduler and introduce new class named
     LeastNetworksScheduler
     This activity includes moving the common methods to parent class named
     DHCPScheduler and inheriting those methods in ChanceScheduler and
     LeastNetworksScheduler classes.

   - LeastNetworksScheduler
     This class will have the implementation to get the dhcp agents which is
     hosting least number of networks. Subsequently it will schedule the
     respective DHCP namespace services on those minimally loaded DHCP agents.


Dependencies
============
Will reuse few methods from current scheduler classes.

Testing
=======
The code will be covered with unit tests.

Tempest Tests
-------------
None

Functional Tests
----------------
Will be added.

API Tests
---------
None

Documentation Impact
====================
Current documentation will have to be enhanced to add the content specific to
DHCP service load balancing scheduler.

User Documentation
------------------
Scheduler section of Openstack Configuration Reference document needs to be
modified.

Developer Documentation
-----------------------
None

References
==========
The code has been submitted for review at the below link
https://review.openstack.org/#/c/137017
