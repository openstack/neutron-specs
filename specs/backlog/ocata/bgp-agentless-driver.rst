..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================================
Adding agent-less driver support for neutron-dynamic-routing project
====================================================================

[RFE] neutron_dynamic_routing project is bound to rpc driver closely
  https://bugs.launchpad.net/neutron/+bug/1611632

Problem Description
===================

The current module `BgpPlugin.py <https://github.com/openstack/neutron-dynamic-routing/blob/master/neutron_dynamic_routing/services/bgp/bgp_plugin.py#L41>`_ is bound to RPC driver so closely, so it is not
working well with the agent-less driver. Another problem is the neutron-dynamic-routing
project does not follow the service provider registration mechanism. Currently, for all
advanced service features, all projects are using service provider to load the driver,
so from this view, the `neutron-dynamic-routing <https://github.com/openstack/neutron-dynamic-routing/blob/master/neutron_dynamic_routing>`_ project needs to be refactored.


Proposed Change
===============

Refactor the current neutron-dynamic-routing code to make sure all agent/agent-less
drivers work well.

    1. Make BgpPlugin.py as an overall interface to load different BGP drivers. The
    service name will be DYNAMIC_ROUTING.

    2. Add a new configuration file neutron_dynamic_routing.conf to specify the
    service_providers, currently, the default service provider is RPC agent driver.

    3. The existing bgp_dragent.ini will be used by the RPC driver such as RYU driver and
    quagga driver.

    4. Create a new directory to locate the BgpRpcPluginDriver. The BgpRpcPluginDriver will
    do what the current BgpPlugin did, for example, setup rpc and register callback.


Configuration file impact
--------------------------

Add a new neutron_dynamic_routing.conf to specify the service_providers.

    service_provider = DYNAMIC_ROUTING:PRCDriver:neutron_dynamic_routing.services.bgp.service_drivers.rpc_bgp.BgpRpcPluginDriver:default

Behaviour Changes
-----------------

Currently, for devstack, the dynamic routing agent (DR agent) is started by default. After the code refactor, the DR agent is
only started when the RPC service provider is enabled. But there will not be any behavioural changes because the
RPC back-end which is the current behaviour will remain as it is.

References
==========

https://github.com/openstack/neutron-lbaas/blobl/master/neutron_lbaas/services/loadbalancer/plugin.py
