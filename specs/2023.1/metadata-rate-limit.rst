..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================
Metadata rate limit
===================

Launchpad Bug:
https://bugs.launchpad.net/neutron/+bug/1989199

The metadata agents currently do not limit the number of requests they try to
process. Mis-behaved instances repetitively querying the metadata endpoint can
incur a high load on services above the metadata-agents (Nova and Neutron).

Problem Description
===================

Platform administrators would benefit from being able to rate-limit requests
handled by metadata in order to protect other OpenStack components from DoS.

* Rate-limiting should be configurable via the config files
* Requests should be rate-limited by source IP
* The default settings for the rate-limiting should not impact the normal
  provisioning of an instance (eg. through cloud-init)

Proposed Change
===============

The metadata agent (both in the case of OVN and OVS/linuxbridge) receive
instances requests through an haproxy process it configures.

We could leverage haproxy's stick-tables to implement rate-limiting, similar
to what is done in this haproxy configuration example [1]_. Concretely,
requests coming-in from IPs that have a request rate above a configured
threshold would receive 429s from haproxy, without hitting the metadata-agent.

As far as configuration is concerned, we would introduce some new directives
to enable the rate-limiting and to configure the thresholds and time-window
sizes.

In order to accommodate some events that normally occur during the life of an
instance (for example: cloud-init, periodic refresh of the network-metadata),
we could make the rate-limit burstable, for example: limit the request rate
to 30 over 60s and to 10 over 5s. This could be implemented by creating two
stick-tables holding the request rates over a long base window and over a
short burst window and by denying requests coming from an IP when its request
rate crosses either the base or the burst threshold.

Impact
======

Configuration impact
--------------------

Add new configuration directives to enable and control the rate-limiting:

* rate_limit_enabled (bool): Whether or not to enable request rate-limiting.
* base_window_duration (seconds): Duration of the base window
* burst_window_duration (seconds): Duration of the burst window
* base_query_rate_limit (short): Limit of the query rate expressed over the
  duration of the base window.
* burst_query_rate_limit (short): Limit of the query rate expressed over the
  duration of the burst window.


Metadata's haproxy impact
-------------------------

* Add two backends meant to hold the stick-tables storing ``http_req_rate``
  per ip for the base window and for the burst window.
* In the listener, deny requests with 429 if ``src_http_req_rate`` is greater
  than the configured threshold of either the base or burst limits.
* In the listener, track requests in both the base and burst stick-tables.


Implementation
==============

Assignee(s)
-----------

* Guillaume Espanel <guillaume.espanel@gmail.com>

Testing
=======

* Unit tests to ensure haproxy's configuration is built according to the
  config parameters.

References
==========

.. [1] HAProxy Rate Limiting: Four Examples, July 30, 2019
   https://www.haproxy.com/blog/four-examples-of-haproxy-rate-limiting/#sliding-window-rate-limiting