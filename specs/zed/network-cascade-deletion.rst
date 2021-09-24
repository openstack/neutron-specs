..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
Add Possibility for Cascade Deletion of Neutron Networks
========================================================

https://bugs.launchpad.net/neutron/+bug/1870319

When creating a network, a user needs to create other resources such
as subnets and ports depending on their needs. They would need to make
numerous API calls to the Neutron server to achieve this, and that's okay.
Deleting the network doesn't need to be that much of a hassle, though.
This spec explores a way to extend the delete API command to include
an argument to cascade delete all corresponding ports and subports
upon the deletion of a network.

Problem Description
===================

Currently, deleting a network with many ports requires many API calls
to the Neutron server as the user needs to list all ports and delete
them before requesting the deletion of the network itself. Even more
requests are required when the network is plugged to a router(s) or has
trunk ports as those also need to be removed before network can be deleted.
A simple ``openstack network delete <network name or ID>`` as prescribed
in the OpenStackClient (OSC) to delete a given network does not work if
there is one or more ports still in use on the network.
In other cases, e.g. when Kuryr [1]_ is used, it puts a lot of load on
Neutron's client-side. This is what Kuryr currently needs to do to delete
"namespace":

* Get the ACTIVE ports in the subnet.
* Get the trunks.
* For each ACTIVE port, obtain what trunk it is attached to
  and call Neutron to detach it.
* Remove the ports (one by one; there is no bulk deletion).
* Get the DOWN ports in the given subnet.
* Remove the ports (one by one; there is no bulk deletion).
* Detach the subnet from the router.
* Remove the network (which will also remove the subnet).

Proposed Change
===============

The goal of this spec is to add a new API to allow for the cascade
deletion of a given network and all the resources that belong to that
network, similarly to Octavia's ``loadbalancer delete --cascade`` option [2]_.
When the proposed network cascade deletion API call is sent to the
Neutron server, it is expected to find and clean:

* All router interfaces that belong to the subnets in that network,
* All trunk ports whose parent ports are in that network. It will also
  need to find and delete all the subports in these trunks,
* Remove all ports in that network, this includes especially ports owned by
  nova compute, attached to VMs; ports owned by ``DEVICE_OWNER_DHCP``,
  ``DEVICE_OWNER_DISTRIBUTED`` or ``DEVICE_OWNER_AGENT_GW`` can be
  omitted as they will have been already automatically removed.

After all this cleaning is done, the network will be finally deleted.

REST API Impact
---------------

The new option will be added to the ``DELETE /v2.0/networks/{network_id}``
API [3]_.

The new option will be called ``cascade``, and will be a boolean value.

If this new option is set to ``True``, Neutron will remove all the
resources that need to be cleaned to successfully remove the given network.

The main concern regarding the proposed API is the return code in case
the deletion of some resources fails and thus the request is only
partially completed. There are a few possible options in such a case:

1. Return ``HTTP 207 Multi-status`` [4]_ and in the response body,
   return a list of actions on various resources and their statuses,
   for example:

   .. code-block:: python

     {
         "data": [
             {
                 "message": "Success",
                 "resource": {
                     "name": "port",
                     "id": "40ac1143-1488-42df-bf85-b0fde7598c97"
                 },
                 "status": 200
             },
             {
                 "message": "Forbidden",
                 "resource": {
                     "name": "trunk",
                     "id": "cb4acbfa-190a-4ad2-9227-ea8595b59b19",
                 },
                 "status": 403
             },
             {
                 "message": "Network in use.",
                 "resource": {
                     "name": "network",
                     "id": "a7ebad6d-fd63-4024-b7db-8c30f5f9dea3"
                 },
                 "status": 409
             }
         ]
     }

   In this case, Neutron can always stop on first failure, so the response
   body will always contain only one or two resources which failed during
   the request: the network and one of the other resources that had to be
   cleaned in cascade but, for some reason, failed.

2. Stop processing the cascade request on first failure and always
   return ``HTTP 409 Conflict`` with detailed information about what
   went wrong, e.g. "Port <port_id> could not be deleted. Reason: <message>".

3. Make this Neutron API asynchronous. In this case, the return code would
   only tell if the cascade delete request was accepted and will be
   processed by the server, or not. Later, the user would be able to
   periodically check if the network is already deleted or not yet.
   This is how Octavia implemented the cascade deletion of the
   loadbalancer [2]_. But it would break the documented Neutron
   API behaviour [4]_.

The proposal of this spec is to implement third solution from the ones mentioned
above and make this Neutron API asynchronous.


Data Model Impact
-----------------

Network's ``status`` field will have new possible value ``DELETING`` which will
indicate that network is in the middle of the asynchronous deletion.

To avoid execution of several DB transactions at once, e.g. when one worker will
proceed with asynchronous cascade deletion of the network and all its resources
and other one would try to update one of the ports/subnets it will be needed to
add check of the network's status for all CREATE/UPDATE/DELETE operations for
ports and subnets as well as for attach/detach subnets from the routers.


Security Impact
---------------

None


Performance Impact
------------------

Finding and removing all the resources to be cascade deleted will be slow,
so the cascade deletion of a network with many resources may take
a long time.
To minimize that time, we can think of a couple of optimizations like:

* Implementing bulk port deletion,
* Removing a port from a trunk automatically if it is the trunk's subport.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
  Sharon Koech <skoech@protonmail.ch> (IRC: skoech)

Work Items
----------

* REST API update.
* ML2 plugin update.
* CLI update.
* Documentation.
* Tests and CI related changes.


Testing
=======

* Unit Tests.
* API tests.


Documentation Impact
====================

User Documentation
------------------

The new API option must be documented in the Neutron api-ref document.


References
==========

.. [1] https://docs.openstack.org/kuryr/latest/
.. [2] https://docs.openstack.org/api-ref/load-balancer/v2/?expanded=remove-a-load-balancer-detail#remove-a-load-balancer
.. [3] https://docs.openstack.org/api-ref/network/v2/index.html?expanded=delete-network-detail#networks
.. [4] https://docs.openstack.org/api-ref/network/v2/index.html#synchronous-versus-asynchronous-plug-in-behavior
