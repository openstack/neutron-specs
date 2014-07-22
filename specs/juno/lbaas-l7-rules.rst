==========================================
LBaaS Layer 7 rules
==========================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/lbaas-l7-rules

Layer 7 switching takes its name from the OSI model, indicating that the device
switches requests based on layer 7 (application) data. Layer 7 switching is
also known as "request switching", "application switching", and
"content based routing".
A layer 7 switch presents to the outside world a "virtual server" that accepts
requests on behalf of a number of servers and distributes those requests based
on policies that use application data to determine which server should service
which request. This allows for the application infrastructure to be specifically
tuned/optimized to serve specific types of content. For example, one server can
be tuned to serve only images, another for execution of server-side scripting
languages like PHP and ASP, and another for static content such as HTML, CSS and
JavaScript.


Problem description
===================

Use Cases:

 1. Redirect traffic to a Pool that supports static content (HTML, CSS)
 2. Redirect traffic to a Pool that serves images (jpg, png, etc)

Proposed change
===============

Extend the LBaaS API and support Layer 7 switching.

L7 Entities:
 1. L7Rule - Set of attributes that defines which part of the request should
    be matched and how it should be matched.
 2. L7Policy - A collection of L7Rules. Holds the action that should
    be performed when the rules are matched.(Redirect to Pool, Redirect to URL,
    Reject). L7Policy holds a Listener id, so a Listner can evaluate a collection
    of L7Policies. L7Policy will return 'true' when all of the L7Rules that
    belong to this L7Policy are matched. L7Policies under a specific Listener
    are ordered and the first l7Policy that returns a match will be executed.
    When none of the policies match the request gets forwarded to
    listener.default_pool_id

Alternatives
------------

None.

Data model impact
-----------------

Model::

 +--------------------+        +--------------------+
 | Listener           |        | L7Policy           |
 +--------------------+        +--------------------+
 |                    |        |                    |
 |   id               |        |   id               |
 |   other attributes +--------+   action           |
 |                    |        |   pool id          |
 |                    |        |   redirect url     |
 |                    |        |   listener id      |
 +--------------------+        |   index            |
                               |                    |
                               |                    |
                               +-----------------+--+
                                                 |
                                                 |
                              +------------------+--+
                              | L7Rule              |
                              +---------------------+
                              |                     |
                              |  id                 |
                              |  l7 policy id       |
                              |  type               |
                              |  compare type       |
                              |  key                |
                              |  value              |
                              |                     |
                              +---------------------+

Two new entities are introduced: L7Rule and L7Policy
The L7Policy is a container of L7Rules.
The L7Policy contains a reference to a Listener


1. L7Rule object Data Model.


  +----------------+--------------+------+-----+---------+
  | Field          | Type         | Null | Key | Default |
  +================+==============+======+=====+=========+
  | id             | string(36)   | NO   | PRI |         |
  +----------------+--------------+------+-----+---------+
  | l7_policy_id   | string(36)   | NO   | FK  |         |
  +----------------+--------------+------+-----+---------+
  | type           | Enum (*)     | NO   |     |         |
  +----------------+--------------+------+-----+---------+
  | compare_type   | Enum (*)     | NO   |     |         |
  +----------------+--------------+------+-----+---------+
  | key            | string(36)   | NO   |     |         |
  +----------------+--------------+------+-----+---------+
  | value          | string(36)   | YES  |     |         |
  +----------------+--------------+------+-----+---------+

 * type values

   - Hostname
   - Path
   - FileType: This is the file extension. Examples: txt, jpg, png, xls
               A rule that is looking for text files will look like:
               type = FileType, compare_type=EqualTo, value = txt
   - Header
   - Cookie: This is the value of a specific cookie
             A rule that is looking for a cookie named 'department'
             with the value starting with 'finance-' will look like:
             type = Cookie, compare_type=StartsWith, key = department
             value = finance-

  * compare_type values

   - Regexp
   - StartsWith
   - EndsWith
   - Contains
   - EqualTo
   - GreaterThan
   - LessThan


2. L7Policy object Data Model.

  +----------------+--------------+------+-----+---------+
  | Field          | Type         | Null | Key | Default |
  +================+==============+======+=====+=========+
  | id             | string(36)   | NO   | PRI |         |
  +----------------+--------------+------+-----+---------+
  | listener_id    | string(36)   | NO   | FK  |         |
  +----------------+--------------+------+-----+---------+
  | action         | Enum (*)     | NO   |     |         |
  +----------------+--------------+------+-----+---------+
  | pool_id        | string(36)   | YES  |     |         |
  +----------------+--------------+------+-----+---------+
  | redirect_url   | string(256)  | YES  |     |         |
  +----------------+--------------+------+-----+---------+
  | index          | int          | NO   |     |         |
  +----------------+--------------+------+-----+---------+

  * action: [Reject,RedirectToURL,RedirectToPool]
  * If action is RedirectToURL redirect_url can not be null
  * If action is RedirectToPool pool_id can not be null
  * Index

   - If total policies for this listener is less than index, append to end of
     list.
   - Index numbering starts with 0
   - If policy with same index number exists, insert the new policy at that
     index number and increment all policy indexes for this listener with an
     equal or higher index value.
   - Not specifying an index appends the policy to the list.

REST API impact
---------------
l7rule-create    Create a L7Rule for a given tenant.

Request

    POST /v2.0/l7rules
    Accept: application/json

.. code-block:: javascript

    {
      "l7rule":{
        "l7_policy_id": "6b96ff0cb17a4b859e1e575d221683c5",
        "type":"Header",
        "compare_type":"StartsWith",
        "key":'department',
        "value":"HR"
      }
    }


Response

.. code-block:: javascript

    {
      "l7rule":{
      "id": "6b96ff0cb17a4b859e1e575d221683d7",
      "l7_policy_id": "6b96ff0cb17a4b859e1e575d221683c5",
      "type":"Header",
      "compare_type":"StartsWith",
      "key":'department',
      "value":"HR",
      "tenant_id":"6b96ff0cb17a4b859e1e575d2216845"
      }
    }

l7rule-show    Show information of a given L7Rule.

Request

    GET /v2.0/l7rules/6b96ff0cb17a4b859e1e575d221683d7
    Accept: application/json

Response

.. code-block:: javascript

    {
      "l7rule":{
      "id": "6b96ff0cb17a4b859e1e575d221683d7",
      "l7_policy_id": "6b96ff0cb17a4b859e1e575d221683c5",
      "type":"Header",
      "compare_type":"StartsWith",
      "key":'department',
      "value":"HR"
      "tenant_id":"6b96ff0cb17a4b859e1e575d2216845"
      }
    }

l7rule-delete    Delete a given L7Rule.

Request

    DELETE /v2.0/l7rules/6b96ff0cb17a4b859e1e575d221683d7
    Accept: application/json


l7policy-create    Create a L7Policy for a given tenant.

    POST /v2.0/l7policies
    Accept: application/json

.. code-block:: javascript

    {
      "l7policy":{
      "listener_id": "6b96ff0cb17a4b859e1e575d221683c5",
      "action":"RedirectToPool",
      "pool_id":6b96ff0cb17a4b859e1e575d22168399,
      "index": 2
      }
    }


Response

.. code-block:: javascript

    {
      "l7policy":{
      "id": "6b96ff0cb17a4b859e1e575d221683d7",
      "listener_id": "6b96ff0cb17a4b859e1e575d221683c5",
      "action":"RedirectToPool",
      "pool_id":6b96ff0cb17a4b859e1e575d22168399,
      "tenant_id":"6b96ff0cb17a4b859e1e575d2216845",
      "index": 2
      }
    }

l7policy-show    Show information of a given L7Policy.

Request

    GET /v2.0/l7policies/6b96ff0cb17a4b859e1e575d221683d7
    Accept: application/json

Response

.. code-block:: javascript

    {
      "l7policy":{
      "id": "6b96ff0cb17a4b859e1e575d221683d7",
      "listener_id": "6b96ff0cb17a4b859e1e575d221683c5",
      "action":"RedirectToPool",
      "pool_id":6b96ff0cb17a4b859e1e575d22168399,
      "tenant_id":"6b96ff0cb17a4b859e1e575d2216845",
      "index": 2
      }
    }

l7policy-delete    Delete a given L7Policy.

Request

    DELETE /v2.0/l7policies/6b96ff0cb17a4b859e1e575d221683d7
    Accept: application/json


Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

None.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~avishayb

Other contributors:
  **TBD**

Work Items
----------

* REST API
* DB Schema
* LBaaS plugin and driver API
* CLI update


Dependencies
============

* Depends on the new LBaaS model https://review.openstack.org/#/c/89903/


Testing
=======

* REST API and attributes validation tests
* DB mixin and schema tests
* LBaaS Plugin with mocked driver end-to-end tests
* Specific driver tests for each existing driver supporting L7 switching
* Tempest tests
* CLI tests


Documentation Impact
====================

* Neutron API should be modified with L7Rule and L7Policy entities
* Neutron CLI should be modified with L7Rule and L7Policy entities


References
==========

https://wiki.openstack.org/wiki/Neutron/LBaaS/l7