================
Agent management
================

In a typical OpenStack Networking deployment, some agents run on network
or compute nodes, such as ``neutron-dhcp-agent``, ``neutron-ovs-agent``,
and ``neutron-l3-agent``. This extension enables administrators
(enforced by the policy engine) to view status and update attributes for
agents. Updating agent management API attributes affects operations of
other components, such as OpenStack Networking schedulers. For example,
administrators can disable a specified agent so that OpenStack
Networking schedulers do not schedule resources to it.

For how to use agent management extension and OpenStack Networking
schedulers feature, see the *OpenStack Cloud Administrator Guide*.

**GET** /agents Lists agents that report their status to Networking server.

**GET** /agents/*``agent_id``* Shows details for a specified agent.

**PUT** /agents/*``agent_id``* Updates the admin status and description for the agent.

**DELETE** /agents/*``agent_id``* Deletes a specified agent.

List agents
~~~~~~~~~~~

**GET** /agents

Lists agents that report their status to OpenStack Networking server.

Normal Response Code: 200

This operation does not require a request body.

This operation returns a response body. The default policy behavior is
that non-admin user won't be able to see any agent in the response when
this call is invoked

**Example List agents: JSON request**

.. code::

    GET /v2.0/agents HTTP/1.1
    Host: controlnode:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: c52a1b304fec4ca0ac85dc1741eec6e2


**Example List agents: JSON response**

.. code::

    {
       "agents":[
          {
             "binary":"neutron-dhcp-agent",
             "description":null,
             "admin_state_up":false,
             "heartbeat_timestamp":"2013-03-26T09:35:13.000000",
             "alive":false,
             "id":"af4567ad-c2e6-4311-944d-22efc12f89af",
             "topic":"dhcp_agent",
             "host":"HostC",
             "agent_type":"DHCP agent",
             "started_at":"2013-03-26T09:35:01.000000",
             "created_at":"2013-03-26T09:35:01.000000",
             "configurations":{
                "subnets":2,
                "use_namespaces":true,
                "dhcp_driver":"neutron.agent.linux.dhcp.Dnsmasq",
                "networks":2,
                "dhcp_lease_time":120,
                "ports":3
             }
          }
       ]
    }


Show agent details
~~~~~~~~~~~~~~~~~~


**GET** /agents/*``agent_id``*

Shows details for a specified agent.

Normal Response Code: 200

Error Response Codes:NotFound (404) if not authorized or the agent does
not exist

This operation returns information for the given agent.

This operation does not require a request body.

This operation returns a response body. The body contents depend on the
agent's type.

**Example Show agent details: JSON request**

.. code::

    GET /v2.0/agents/af4567ad-c2e6-4311-944d-22efc12f89af HTTP/1.1
    Host: controlnode:9696
    User-agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: a54d6fdda41341f892150e2aaf648f0d



**Example Show agent details: JSON response**

.. code::

    {
       "agent":{
          "binary":"neutron-dhcp-agent",
          "description":null,
          "admin_state_up":false,
          "heartbeat_timestamp":"2013-03-26T09:35:13.000000",
          "alive":false,
          "id":"af4567ad-c2e6-4311-944d-22efc12f89af",
          "topic":"dhcp_agent",
          "host":"HostC",
          "agent_type":"DHCP agent",
          "started_at":"2013-03-26T09:35:01.000000",
          "created_at":"2013-03-26T09:35:01.000000",
          "configurations":{
             "subnets":2,
             "use_namespaces":true,
             "dhcp_driver":"neutron.agent.linux.dhcp.Dnsmasq",
             "networks":2,
             "dhcp_lease_time":120,
             "ports":3
          }
       }
    }



Update agent
~~~~~~~~~~~~

**PUT**

/agents/*``agent_id``*

Updates the admin status and description for a specified agent.

Normal Response Code: 200

Error Response Codes: BadRequest (400) if something other than
description or admin status is changed, NotFound (404) if not authorized
or the agent does not exist

This operation updates the agent's admin status and description.

This operation requires a request body.

This operation returns a response body.

**Example Update agent: JSON request**

.. code::

    PUT /v2.0/agents/af4567ad-c2e6-4311-944d-22efc12f89af HTTP/1.1
    Host: controlnode:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: 4cbb09e780434b249ff596d6979fd8fc
    Content-Length: 38{
        "agent": {
            "admin_state_up": "False"
        }
    }



**Example Update agents: JSON response**

.. code::

    {
       "agent":{
          "binary":"neutron-dhcp-agent",
          "description":null,
          "admin_state_up":false,
          "heartbeat_timestamp":"2013-03-26T09:35:13.000000",
          "alive":false,
          "id":"af4567ad-c2e6-4311-944d-22efc12f89af",
          "topic":"dhcp_agent",
          "host":"HostC",
          "agent_type":"DHCP agent",
          "started_at":"2013-03-26T09:35:01.000000",
          "created_at":"2013-03-26T09:35:01.000000",
          "configurations":{
             "subnets":2,
             "use_namespaces":true,
             "dhcp_driver":"neutron.agent.linux.dhcp.Dnsmasq",
             "networks":2,
             "dhcp_lease_time":120,
             "ports":3
          }
       }
    }



Delete agent
~~~~~~~~~~~~

**DELETE**

/agents/*``agent_id``*

Deletes a specified agent.

Normal Response Code: 204

Error Response Codes: NotFound (404) if not authorized or the agent does
not exist

This operation deletes the agent.

This operation does not require a request body.

This operation does not return a response body.

**Example Delete agent: JSON request**

.. code::

    DELETE /v2.0/agents/44002aeb-2817-4cb8-9306-34308b4b40d9 HTTP/1.1
    Host: controlnode:9696
    User-Agent: python-neutronclient
    Content-Type: application/json
    Accept: application/json
    X-Auth-Token: 4cbb09e780434b249ff596d6979fd8fc



**Example Delete agent: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0
    Date: Tue, 26 Mar 2013 12:12:35 GMT



