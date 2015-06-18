..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
LBaaS plugin can delegate VIP allocation to drivers
===================================================

https://bugs.launchpad.net/neutron/+bug/1463594

Currently the neutron lbaas plugin will create a neutron port for a VIP and
then pass that information to the driver to continue creation of a load
balancer.  There are some situations in which a driver will require control
of how the VIP is created.  Work on Octavia has brought the realization that
this feature is needed to allow deployments from different providers.

Problem Description
===================

Since the neutron lbaas plugin creates a neutron port as the VIP for drivers
to use, drivers that need to do more complex VIP allocation or drivers that
require the VIP allocation to be delayed further in the workflow will have to
do some untenable workarounds to make work.

Some examples from a deployers point of view are:

* Assume an environment in which cells in nova are being used and in which the
  VIP is going to be the IP that the nova instance is allocated.  Since cells
  are being used, this may mean that the nova scheduler will decide which
  segment and subnet the VIP should be allocated from.  If the neutron port is
  created before the instance is created, then the segment and/or subnet the
  IP was allocated from may cause conflicts with the nova scheduler.  The
  cells and network segment are explained and discussed in the RFE [1].

* The VIP allocation may need some external service to do the actual allocation
  or extra pre-setup of the VIP, in which case it can't be created in the
  neutron lbaas plugin and would need to be left to the driver to create.

Proposed Change
===============

A non-abstract method will be added in the driver interface.  This way existing
drivers will not need to be changed unless they want to take advantage of this
new ability.  Before the plugin creates the neutron port for the VIP, it will
query the driver and determine whether that new method has been implemented.
If it has been implemented, the driver will call that method and will not
create the neutron port for the VIP.  If it has not been implemented then the
normal port creation and driver call will take place.  Implementing it this way
will not leak implementation details to the user or the deployer as it becomes
just a decision for the driver implementer.

One minor issue this may cause is if the driver implements the new method
asynchronously.  The method will return without any information of the VIP as
it probably has not been allocated.  This will probably require a thread to
run in the background checking for the allocation and then updating the neutron
lbaas database.  The end user will not see an IP for the VIP immediately upon
return of the create load balancer call.  The population of that field will
end up being delayed, much like how a nova instance does not immediately have
an IP assigned.

I do think this is an acceptable change to the API behavior
because when the IP is immediately returned, its still not a usable IP until
the load balancer has completed provisioning.

References
==========

[1] https://bugs.launchpad.net/neutron/+bug/1458890
