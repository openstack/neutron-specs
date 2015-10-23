================
Agent schedulers
================

The agent scheduler extensions schedule resources among agents.

The agent scheduler feature consist of several agent scheduler
extensions. In Havana, the following extensions are available.

-  DHCP agent scheduler (``dhcp_agent_scheduler``)

-  L3 agent scheduler (``l3_agent_scheduler``)

-  load balancer agent scheduler (``lbaas_agent_scheduler``)

In Grizzly, the DHCP agent scheduler and the L3 agent scheduler features
are provided by a single extension named the agent scheduler
(``agent_scheduler``). In Havana, this extension is split into the DHCP
agent scheduler and the L3 agent scheduler extensions. The load balancer
agent scheduler extension was introduced in Havana.

DHCP agent scheduler (``dhcp_agent_scheduler``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The DHCP agent scheduler extension enables administrators to assign DHCP
servers for Neutron networks to given Neutron DHCP agents, and retrieve
mappings between Neutron networks and DHCP agents. This feature is
implemented on top of Agent Management extension.

**GET** /agents/*``agent_id``*/dhcp-networks

Lists networks that the specified DHCP agent hosts.

**GET** /networks/*``network_id``*/dhcp-agents

Lists DHCP agents that host a specified network.

**POST** /agents/*``agent_id``*/dhcp-networks

Schedules the network to that the specified DHCP agent.

**DELETE** /agents/*``agent_id``*/dhcp-networks/*``network_id``*

Removes the network from that the specified DHCP agent.

List networks hosted by a DHCP agent
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**GET** /agents/*``agent_id``*/dhcp-networks

Lists networks that the specified DHCP agenthosts.

Normal response Code: 200

Error response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example List networks hosted by on DHCP agent: JSON request**

.. code::

    GET /v2.0/agents/d5724d7e-389d-4ba0-8d00-fc673a147bfa/dhcp-networks HTTP/1.1
    Host: localhost:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: 797f94caf0a8481c893a232cc0c1dfca


**Example List networks hosted by DHCP agent: JSON response**

.. code::

    {
       "networks":[
          {
             "status":"ACTIVE",
             "subnets":[
                "15a09f6c-87a5-4d14-b2cf-03d97cd4b456"
             ],
             "name":"net1",
             "provider:physical_network":"physnet1",
             "admin_state_up":true,
             "tenant_id":"3671f46ec35e4bbca6ef92ab7975e463",
             "provider:network_type":"vlan",
             "router:external":false,
             "shared":false,
             "id":"2d627131-c841-4e3a-ace6-f2dd75773b6d",
             "provider:segmentation_id":1001
          },
          {
             "status":"ACTIVE",
             "subnets":[

             ],
             "name":"net2",
             "provider:physical_network":null,
             "admin_state_up":true,
             "tenant_id":"3671f46ec35e4bbca6ef92ab7975e463",
             "provider:network_type":"local",
             "router:external":false,
             "shared":false,
             "id":"524e26ea-fad4-4bb0-b504-1ad0dc770e7a",
             "provider:segmentation_id":null
          },
          {
             "status":"ACTIVE",
             "subnets":[
                "43671fba-c76b-4c33-bd7e-8bef54145f2f"
             ],
             "name":"mynet1",
             "provider:physical_network":"physnet1",
             "admin_state_up":true,
             "tenant_id":"3671f46ec35e4bbca6ef92ab7975e463",
             "provider:network_type":"vlan",
             "router:external":false,
             "shared":false,
             "id":"cfa65a54-06a8-4f9f-86b0-73c700c02c41",
             "provider:segmentation_id":1000
          }
       ]
    }


List DHCP agents hosted by network
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**GET** /networks/*``network_id``*/dhcp-agents

Lists DHCP agents that hosts a specified network.

Normal response Code: 200

Error response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example List DHCP agents hosted by network: JSON request**

.. code::

    GET /v2.0/networks/2d627131-c841-4e3a-ace6-f2dd75773b6d/dhcp-agents HTTP/1.1
    Host: localhost:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: cc0f378bdf1545fb8dea2120c89eb532



**Example List DHCP agents hosted by network: JSON response**

.. code::

    {
       "agents":[
          {
             "binary":"neutron-dhcp-agent",
             "description":null,
             "admin_state_up":true,
             "heartbeat_timestamp":"2013-03-27T00:24:01.000000",
             "alive":false,
             "topic":"dhcp_agent",
             "host":"HostC",
             "agent_type":"DHCP agent",
             "created_at":"2013-03-26T23:54:20.000000",
             "started_at":"2013-03-26T23:54:20.000000",
             "id":"d5724d7e-389d-4ba0-8d00-fc673a147bfa",
             "configurations":{
                "subnets":2,
                "use_namespaces":true,
                "dhcp_driver":"neutron.agent.linux.dhcp.Dnsmasq",
                "networks":2,
                "dhcp_lease_time":120,
                "ports":5
             }
          }
       ]
    }



Schedule network to DHCP agent
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**POST** /agents/*``agent_id``*/dhcp-networks

Schedules the network to that the specified DHCP agent.

Normal response Code: 201

Error response Codes: Unauthorized (401), Forbidden (403), Conflict
(409) if the network is already hosted by that the specified DHCP agent,
NotFound(404) when the specified agent is not a valid DHCP agent.

This operation requires a request body.

This operation returns a ``null`` body.

**Example Schedule network: JSON request**

.. code::

    POST /v2.0/agents/d5724d7e-389d-4ba0-8d00-fc673a147bfa/dhcp-networks.json HTTP/1.1
    Host: localhost:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: d88f7af21ee34f6c87e23e46cf3f986d
    Content-Length: 54

    {"network_id": "1ae075ca-708b-4e66-b4a7-b7698632f05f"}


**Example Schedule network: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8
    Content-Length: 4
    Date: Wed, 27 Mar 2013 01:22:46 GMT

    null


Remove network from DHCP agent
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**DELETE** /agents/*``agent_id``*/dhcp-networks/*``network_id``*

Removes the network from that the specified DHCP agent.

Normal response Code: 204

Error response Codes: Unauthorized (401), Forbidden (403), NotFound
(404), Conflict (409) if the network is not hosted by that the specified
DHCP agent.

This operation does not require a request body.

This operation does not return a response body.

**Example Remove network from DHCP agent: JSON request**

.. code::

    DELETE /v2.0/agents/d5724d7e-389d-4ba0-8d00-fc673a147bfa/dhcp-networks/1ae075ca-708b-4e66-b4a7-b7698632f05f.json HTTP/1.1
    Host: localhost:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: 7ae91cde8f504031be5a2cd5b99d4fe9


L3 agent scheduler (``l3_agent_scheduler``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The L3 agent scheduler extension allows administrators to assign Neutron
routers to Neutron L3 agents, and retrieve mappings between Neutron
routers and L3 agents. This feature is implemented on top of Agent
Management extension.

**GET** /agents/*``agent_id``*/l3-routers

Lists routers that the specified L3 agent hosts.

**GET** /routers/*``router_id``*/l3-agents

Lists L3 agents that hosts a specified router.

**POST** /agents/*``agent_id``*/l3-routers

Schedules the router to that the specified L3 agent.

**DELETE** /agents/*``agent_id``*/l3-routers/*``router_id``*

Removes the router from that the specified L3 agent.

List routers hosted by an L3 agent
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**GET** /agents/*``agent_id``*/l3-routers

Lists routers that the specified L3 agent hosts.

Normal response Code: 200

Error response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example List routers hosted by L3 agent: JSON request**

.. code::

    GET /v2.0/agents/fa24e88e-3d2f-4fc2-b038-5fb5be294c03/l3-routers.json HTTP/1.1
    Host: localhost:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: 6eeea6e73b68415f85d8368902a32c11



**Example List routers hosted by L3 agent: JSON response**

.. code::

    {
       "routers":[
          {
             "status":"ACTIVE",
             "external_gateway_info":null,
             "name":"router1",
             "admin_state_up":true,
             "tenant_id":"3671f46ec35e4bbca6ef92ab7975e463",
             "routes":[

             ],
             "id":"8eef2388-f27d-4a17-986e-9319a77ccd9d"
          }
       ]
    }



List L3 agents hosted by router
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**GET** /routers/*``router_id``*/l3-agents

Lists L3 agents that hosts a specified router.

Normal response Code: 200

Error response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example List L3 agents hosted by router: JSON request**

.. code::

    GET /v2.0/routers/8eef2388-f27d-4a17-986e-9319a77ccd9d/l3-agents.json HTTP/1.1
    Host: localhost:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: bce63afb1e794c70972a19a7c2d6dcab



**Example List L3 agents hosted by router: JSON response**

.. code::

    {
       "agents":[
          {
             "binary":"neutron-l3-agent",
             "description":null,
             "admin_state_up":true,
             "heartbeat_timestamp":"2013-03-27T00:24:03.000000",
             "alive":false,
             "topic":"l3_agent",
             "host":"HostC",
             "agent_type":"L3 agent",
             "created_at":"2013-03-26T23:54:26.000000",
             "started_at":"2013-03-26T23:54:26.000000",
             "id":"fa24e88e-3d2f-4fc2-b038-5fb5be294c03",
             "configurations":{
                "router_id":"",
                "gateway_external_network_id":"",
                "handle_internal_only_routers":true,
                "use_namespaces":true,
                "routers":0,
                "interfaces":0,
                "floating_ips":0,
                "interface_driver":"neutron.agent.linux.interface.OVSInterfaceDriver",
                "ex_gw_ports":0
             }
          }
       ]
    }



Schedule router to L3 agent
^^^^^^^^^^^^^^^^^^^^^^^^^^^

**POST** /agents/*``agent_id``*/l3-routers

Schedules one router to that the specified L3 agent.

Normal response Code: 201

Error response Codes: Unauthorized (401), Forbidden (403), Conflict
(409) if the router is already hosted, NotFound (404) if the specified
agent is not a valid L3 agent.

This operation requires a request body.

This operation returns a ``null`` body.

**Example Schedule router: JSON request**

.. code::

    POST /v2.0/agents/fa24e88e-3d2f-4fc2-b038-5fb5be294c03/l3-routers.json HTTP/1.1
    Host: localhost:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: d88f7af21ee34f6c87e23e46cf3f986d
    Content-Length: 54

    {"router_id": "8eef2388-f27d-4a17-986e-9319a77ccd9d"}



**Example Schedule router: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8
    Content-Length: 4
    Date: Wed, 27 Mar 2013 01:22:46 GMT

    null



Remove router from L3 agent
^^^^^^^^^^^^^^^^^^^^^^^^^^^

**DELETE** /agents/*``agent_id``*/l3-routers/*``network_id``*

Removes the router from that the specified L3 agent.

Normal response Code: 204

Error response Codes: Unauthorized (401), Forbidden (403), Conflict
(409) if the router is not hosted by that the specified L3 agent.

This operation does not require a request body.

This operation does not return a response body.

**Example Remove router from L3 agent: JSON request**

.. code::

    DELETE /v2.0/agents/b7d7ba43-1a05-4b09-ba07-67242d4a98f4/l3-routers/8eef2388-f27d-4a17-986e-9319a77ccd9d.json HTTP/1.1
    Host: localhost:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: 2147ef6fe4444f0299b1c0b6b529ff47


Load balancer agent scheduler (``lbaas_agent_scheduler``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The LBaaS agent scheduler extension allows administrators to retrieve
mappings between load balancer pools to LBaaS agents. In Havana, this
extension does not provide an ability to assign load balancer pool to
specific LBaaS agent. Pools are scheduled automatically when created.
This feature is implemented on top of Agent Management extension. The
load balancer agent scheduler extension was introduced in Havana.

**GET** /agents/*``agent_id``*/loadbalancer-pools

Lists pools that the specified LBaaS agent hosts.

**GET** /lb/pools/*``pool_id``*/loadbalancer-agent

Shows an LBaaS agent that hosts a specified pool.

List pools hosted by an LBaaS agent
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**GET** /agents/*``agent_id``*/loadbalancer-pools

Lists pools that the specified LBaaS agent hosts.

Normal response Code: 200

Error response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example List pools hosted by LBaaS agent: JSON request**

.. code::

    GET /v2.0/agents/6ee1df7f-bae4-4ee9-910a-d33b000773b0/loadbalancer-pools.json HTTP/1.1
    Host: localhost:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: 6eeea6e73b68415f85d8368902a32c11



**Example List pools hosted by LBaaS agent: JSON response**

.. code::

    {
        "pools": [
            {
                "admin_state_up": true,
                "description": "",
                "health_monitors": [],
                "health_monitors_status": [],
                "id": "28296abb-e675-4288-9cd0-6c112c720db0",
                "lb_method": "ROUND_ROBIN",
                "members": [],
                "name": "pool1",
                "protocol": "HTTP",
                "provider": "haproxy",
                "status": "PENDING_CREATE",
                "status_description": null,
                "subnet_id": "f8fd83d3-2080-4ab9-9814-391fe7b8a7a4",
                "tenant_id": "54d7b6253c8c4e64862fbd08b3fc08cd",
                "vip_id": null
            }
        ]
    }



Show LBaaS agent that hosts pool
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**GET** /lb/pools/*``pool_id``*/loadbalancer-agent

Shows an LBaaS agent that hosts a specified pool.

Normal response Code: 200

Error response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example Show LBaaS agent that hosts pool: JSON request**

.. code::

    GET /v2.0/lb/pools/28296abb-e675-4288-9cd0-6c112c720db0/loadbalancer-agent.json HTTP/1.1
    Host: localhost:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: bce63afb1e794c70972a19a7c2d6dcab



**Example Show LBaaS agent that hosts pool: JSON response**

.. code::

    {
        "agent": {
            "admin_state_up": true,
            "agent_type": "Loadbalancer agent",
            "alive": true,
            "binary": "neutron-loadbalancer-agent",
            "configurations": {
                "device_driver": "neutron.services.loadbalancer.drivers.haproxy.namespace_driver.HaproxyNSDriver",
                "devices": 0,
                "interface_driver": "neutron.agent.linux.interface.OVSInterfaceDriver"
            },
            "created_at": "2013-10-01 12:50:13",
            "description": null,
            "heartbeat_timestamp": "2013-10-01 12:56:29",
            "host": "ostack02",
            "id": "6ee1df7f-bae4-4ee9-910a-d33b000773b0",
            "started_at": "2013-10-01 12:50:13",
            "topic": "lbaas_process_on_host_agent"
        }
    }



