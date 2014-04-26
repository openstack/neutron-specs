..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Enable JSON support for N1KV REST calls
==========================================

https://blueprints.launchpad.net/neutron/+spec/cisco-n1kv-json-support

Need to enable Cisco N1KV plugin to accept REST API responses in JSON.

Problem description
===================

Currently, Cisco N1KV Neutron plugin and VSM (controller) communicate
using REST APIs. The VSM is capable of returning responses in XML and JSON.
However, the plugin handles only XML responses.

Proposed change
===============

The proposed change is to use the Requests library to support handling of
REST API responses in JSON. The Requests library will replace the httplib2
library currently being used by the plugin.

Alternatives
------------

None

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

None

Performance Impact
------------------

None

Other deployer impact
---------------------

requests library version >= 1.1
(as per neutron requirements.txt)

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  sopatwar

Other contributors:
  abhraut

Work Items
----------

- Replace httplib2 library used in the plugin with Requests library.
- Modify all the methods in n1kvclient to use the Requests library.
- Replace the current XML parsing logic in the response handlers for policy
  profiles to handle JSON responses.

Dependencies
============

None

Testing
=======

Currently, unit tests include coverage of XML responses.
The test code will be modified to handle JSON responses.

Documentation Impact
====================

None

References
==========

None
