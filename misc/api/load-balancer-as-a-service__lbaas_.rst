==================================
Load-Balancer-as-a-Service (LBaaS)
==================================

The LBaaS extension enables OpenStack tenants to load-balance their VM
traffic.

The extension enables you to:

-  Load-balance client traffic from one network to application services,
   such as VMs, on the same or a different network.

-  Load-balance several protocols, such as TCP and HTTP.

-  Monitor the health of application services.

-  Support session persistence.

Concepts
~~~~~~~~

This extension introduces these concepts:

Load balancers
    The primary load-balancing configuration object. Specifies the
    virtual IP address where client traffic is received.

Pools
    A logical set of devices, such as web servers, that you group
    together to receive and process traffic.

    The load-balancing algorithm chooses which member of the pool
    handles new requests or connections that are received on a listener.
    Each listener has one default pool.

Listener
    Represents a single listening port. Defines the protocol and can
    optionally provide TLS termination.

Members
    The application that runs on the back-end server.

Health monitors
    Determines whether or not back-end members of the pool can process a
    request. A pool can have one health monitor associated with it.

    The LBaaS extension supports these types of health monitors:

    -  **PING**. Uses ICMP to ping the members.

    -  **TCP**. Uses TCP to connect to the members.

    -  **HTTP**. Sends an HTTP request to the member.

    -  **HTTPS**. Sends a secure HTTP request to the member.

Session persistence
    Forces connections or requests in the same session to be processed
    by the same member as long as it is active.

    The LBaaS extension supports these types of persistence:

    -  **SOURCE\_IP**. All connections that originate from the same
       source IP address are handled by the same member of the pool.

    -  **HTTP\_COOKIE**. The load-balancing function creates a cookie on
       the first request from a client. Subsequent requests that contain
       the same cookie value are handled by the same member of the pool.

    -  **APP\_COOKIE**. The load-balancing function relies on a cookie
       established by the back-end application. All requests with the
       same cookie value are handled by the same member of the pool.

    Absence of ``session_persistence`` attribute means no session
    persistence mechanism is used.

    When no session persistence is used, the ``session_persistence``
    attribute does not appear in the API response and instead returns
    null.

    You can clear session persistence by sending ``null`` in
    ``session_persistence`` attribute in a listener update request.

Use the LBaaS extension to configure load balancing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You must complete these high-level tasks:

**To use the LBaaS extension to configure load balancing**

#. Create a pool, which is initially empty.

#. Create one or more members in the pool.

#. Create a health monitor.

#. Associate the health monitor with the pool.

#. Create a load balancer object.

#. Create a listener.

#. Associate the listener with the load balancer.

#. Associate the pool with the listener.

#. Optional. If you use HTTPS termination, complete these tasks:

   #. Add the TLS certificate, key, and optional chain to Barbican.

   #. Associate the Barbican container with the listener.

#. Optional. If you use layer-7 HTTP switching, complete these tasks:

   #. Create any additional pools, members, and health monitors that are
      used as non-default pools.

   #. Create a layer-7 policy that associates the listener with the
      non-default pool.

   #. Create rules for the layer-7 policy that describe the logic that
      selects the non-default pool for servicing some requests.

Load balancers
~~~~~~~~~~~~~~

Use the LBaaS extension to create and manage load balancers.

Listeners
~~~~~~~~~

Use the LBaaS extension to create and manage load-balancer listeners.

Pools
~~~~~

Use the LBaaS extension to create and manage load-balancer pools.

Members
~~~~~~~

Use the LBaaS extension to create and manage load-balancer pool members.

Health monitors
~~~~~~~~~~~~~~~

Use the LBaaS extension to create and manage load-balancer health
monitors.
