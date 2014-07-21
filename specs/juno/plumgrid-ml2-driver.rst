..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
PLUMgrid ML2 mechanism driver
=============================

https://blueprints.launchpad.net/neutron/+spec/plumgrid-ml2-driver

* PG-MD : PLUMgrid Mechanism Driver

The purpose of this blueprint is to build an ML2 Mechanism Driver for PLUMgrid
virtual network infrastructure, which proxies RESTful calls from ML2 plugin
of Neutron to PLUMgrid Director(s).

Problem description
===================

PLUMgrid Director(s) requires information of OpenStack Neutron APIs to manage
networking.

In order to recieve such information from neutron service, a new ML2 mechanism
driver is needed to post the data to PLUMgrid Director.

The following sections describe the proposed changes in Neutron, a new ML2
mechanism driver, and make it possible to use OpenStack with PLUMgrid virtual
network infrastructure. The following diagram depicts the OpenStack deployment
with PLUMgrid virtual network infrastructure.

PLUMgrid-OpenStack Topology::

  +-----------------------+              +----------------+
  |                       |              |                |
  |     OpenStack         |              |                |
  |     Controller        |              |                |
  |     Node              |              |                |
  |                       |              |                |
  | +---------------------+              |   PLUMgrid     |
  | |PLUMgrid mechanism   | REST API     |   Director     |
  | |driver               |--------------|                |
  | |                     |              |                |
  +-+--------+-----+------+              +--+----------+--+
             |     |                        |          |
             |     |                        |          |
             |     +--------------+         |          |
             |                    |         |          |
  +----------+---------+      +---+---------+------+   |
  |                    |      |                    |   |
  |      IOvisor       |      |      IOvisor       |   |
  +--------------------+ ---- +--------------------+   |
  | OpenStack compute  |      | OpenStack compute  |   |
  | node 1             |      | node n             |   |
  +----------+---------+      +--------------------+   |
             |                                         |
             |                                         |
             +-----------------------------------------+

As shown in the diagram above, each OpenStack compute node is connected
to PLUMgrid Director, which is responsible for provisioning, monitoring
and troubleshooting of cloud network infrastructures, which includes PLUMgrid
virtual network infrastructure. The Neutron API requests will be proxied to
PLUMgrid Director, then network topology information can be built and deployed.
The compute nodes will have PLUMgrid IOvisor running.

Proposed change
===============
The requirements for ML2 mechanism driver to support PLUMgrid virtual network
infrastructure are as follow:

1. PLUMgrid Director exchanges information with OpenStack controller node by
using REST API. To support this, we need a specific client.

2. OpenStack controller (Neutron configured with ML2 plugin) must be
configured with PLUMgrid Director access credentials.

Sequence flow of events for a sample API call (create_network) is as follow:

::

 create_network
 {
   neutron    ->  ML2_plugin
   ML2_plugin ->  PG-MD
   PG-MD      ->  PLUMgrid Director
   PG-MD      <-- PLUMgrid Director
   ML2_plugin <-- PG-MD
   neutron    <-- ML2_plugin
 }

Alternatives
------------

The alternative would be to continue using the monolithic plugin for
PLUMgrid. In order to simplify the controller and in order to take
advantage of the support for various extensions already available in
ML2 and to avoid replicating code as much as possible and in order to
simplify the maintenance of our code we have decided to make the move
to ML2 and develop a mechanism driver for it.

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

1. Configuration parameters for PLUMgrid mechanism driver
will have to be updated.

Update /etc/neutron/plugins/ml2/ml2_conf_plumgrid.ini, as follow:

::
  [PLUMgridDirector]
  director_server=<PLUMgrid_Director_IP>
  director_server_port=<PLUMgrid_Director_Port>
  username=<PLUMgrid_Director_Admin>
  password=<PLUMgrid_Director_Admin_Password>

Performance Impact
------------------

None

Other deployer impact
---------------------

1. Configuration parameters for PLUMgrid mechanism driver
will have to be updated.

Update /etc/neutron/plugins/ml2/ml2_conf_plumgrid.ini, as follow:

::
  [PLUMgridDirector]
  director_server=<PLUMgrid_Director_IP>
  director_server_port=<PLUMgrid_Director_Port>
  username=<PLUMgrid_Director_Admin>
  password=<PLUMgrid_Director_Admin_Password>


Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fawadkhaliq

Work Items
----------

1. Mechanism driver for PLUMgrid
2. Unit test cases
3. Third party testing CI setup for ML2 PLUMgrid

Dependencies
============

None

Testing
=======

Testing is essential here so we will use unit testing and third party CI
to test the proposed changes. PLUMgrid CI for ML2 driver will be provisioned
and maintained.

Documentation Impact
====================

Documentation needs to be updated to incorporate new deployment model
including all the configuration changes.

References
==========
N/A
