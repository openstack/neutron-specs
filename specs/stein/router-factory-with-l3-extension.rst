..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Registering RouterInfo by L3 extension API
==========================================

Launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/router-factory-with-l3-extension

Currently, most plugin implementations related to L3 override the *L3NATAgent*
class itself for their own logic since there is no proper interface to
extend the *RouterInfo* class. This adds unnecessary complexity for developers
who just want to extend the agent mechanism instead of the whole RPC related
to L3 functionalities.

This spec introduces the *RouterFactory* class which acts on the factory for
creating the *RouterInfo* class, and adds a new parameter to the L3 agent
extension API which enables it to dynamically register *RouterInfo* to the
factory. Now plugin developers can use the new extension API for their own
specific router.


Problem Description
===================

Current L3 agent implementation in Neutron consists of two parts in general.
One is to implement an RPC Plugin API from Neutron server, and the other is to
create ports, namespaces, and iptables rules using the data obtained from the
RPC API on the server. To be more specific, the former is the *L3NATAgent*
class and the latter is the *RouterInfo* class.

The problem is that the two parts mentioned are now tightly coupled which means
there is no clear way to extend each part individually. A lot of projects
related to Neutron called networking-* [1]_, [2]_, [3]_ are making new L3
agent classes on their own by extending the *L3NATAgent* class even though
they did not modify the RPC mechanism but only changed the *RouterInfo*
mechanism running on their server.

Also, the current *RouterInfo* class does not have an abstract interface which
makes it harder to extend the class for plugin developers. They have to find
what functions and variables in *RouterInfo* are externally used in the L3
agent to extend the *RouterInfo* behaviors.


Proposed Change
===============

Today, *RouterInfo* is extended in several ways according to specific router
features such as distributed, ha, and distributed + ha. This document proposes
changing the L3 agent to have a new class called ``RouterFactory`` which has
several pre-registered classes to extend the *RouterInfo* class with certain
features. When it comes to creating an actual *RouterInfo* instance, the L3
agent create a new instance from the ``RouterFactory`` following the features
of the router. There is no functional change for existing code.

*L3AgentExtensionAPI* now has a new parameter ``router_factory`` and a new
function ``register_router``. A new abstract class called ``BaseRouterInfo``
will be added. It will declare interfaces that are currently used externally.

An L3 extension can register their own *RouterInfo* class which implements
``BaseRouterInfo`` using the ``register_router`` API which has two parameters.

(1) ``router``: RouterInfo declared in extension which overrides the one
    pre-registered at ``RouterFactory``

(2) ``features``: features of RouterInfo. Currently it should be one of the
    below.

    Features should be a list of strings describing router characteristics, and
    the ordering does not matter since it is interpreted as a *set* internally.
    (``['ha', 'distributed']`` and ``['distributed', 'ha']`` are the same)

    - ``[]``: No feature. (e.g. ``LegacyRouter``)
    - ``['distributed']``: Distributed router. (e.g. ``DvrEdgeRouter``,
                           ``DvrLocalRouter``)
    - ``['ha']``: HA router. (e.g. ``HaRouter``)
    - ``['ha', 'distributed']``: Distributed HA router. (e.g.
                                 ``DvrEdgeHaRouter``, ``DvrLocalRouter``)

    Note that a router with the feature of ``['ha', 'distributed']`` can be
    ``DvrLocalRouter`` when L3 agent mode is not ``dvr_snat`` [4]_.

L3 extensions can override the *RouterInfo* class implemented in the Neutron
codebase when it is initialized using the *initialize* function.


Implementation
==============

Assignee(s)
-----------

* Yang Youseok <ileixe@gmail.com>

Work Items
----------

* Add *RouterInfo* registration in *L3AgentExensionAPI* and pre-register
  existing *RouterInfo* to *L3NATAgent*. [5]_
* Add new exception named *RouterNotFoundInRouterFactory* to *neutron_lib*.
  [6]_


References
==========

.. [1] `networking-odl`:
        https://github.com/openstack/networking-odl/blob/master/networking_odl/l3/l3_odl_v2.py#L47

.. [2] `networking-ovn`:
        https://github.com/openstack/networking-ovn/blob/master/networking_ovn/l3/l3_ovn.py#L50

.. [3] `networking-calico`:
        https://github.com/openstack/networking-calico/blob/master/networking_calico/plugins/calico/plugin.py#L26

.. [4] https://docs.openstack.org/neutron/latest/configuration/l3-agent.html#DEFAULT.agent_mode

.. [5] https://review.openstack.org/#/c/620349/

.. [6] https://review.openstack.org/#/c/620348/
