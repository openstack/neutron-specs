..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


==================================
Neutron LBaaS TLS - Specification
==================================

BP https://blueprints.launchpad.net/neutron/+spec/lbaas-ssl-termination

Terminating TLS connections on the load balancer is a capability
expected from modern load balancers and incorporated into many applications.
This capability enables better certificate management and improved
application based load balancing e.g. cookie-based persistency,
L7 Policies appliance, etc.


Problem description
===================

No TLS offloading capability is available for Neutron LBaaS


Proposed change
===============

*Note: This document is referencing to the new LBaaS objects model
proposed at https://review.openstack.org/#/c/89903*

*Note: This document does not consider flavors framework proposed at
https://review.openstack.org/#/c/90070
Before the flavors framework is in place, specific back-end driver which
does not support TLS capabilities should throw an exception stating a lack
of TLS support once it gets request for listener with TLS configuration.
This document specifies a "core" feature set that every back-end implementing
TLS capabilities must comply.
TLS capabilities of various back-end implementations
may differ in future releases, thus flavors aspect should definitely be part
of TLS capabilities specification*

*Note: Horizon project aspect is not a part of this specification.*

* Tenant will manage his TLS certificates using Barbican.
  Certificates will be stored in Barbican secure containers.
* Barbican is in charge of containers life cycle management,
  containers classification and validation.
  LBaaS TLS requires a specific container type (TLS).
  Only container of this type will be listed to the tenant for selection
  while configuring listener's TLS containers to use.
  Any invalid container usage will raise an error.
* Barbican will also manage list of interested consumers for each container.
  See spec at https://review.openstack.org/#/c/99516

  Neutron LBaaS (a consumer, according to Barbican's terminology) will not
  use a regular GET request for container resource in order to get the
  container. Instead, it will use a POST request to container's consumers
  resource (http://admin-api/v1/containers/{container-uuid}/consumers) with
  following info:

  {
    "type": "LBaaS",
    "URL": "https://lbaas.myurl.net/loadbalancers/<lb_id>/"

  }

  **Note:**
  We might want to use specific Listener URL instead of
  Loadbalancer's one, like
  "https://lbaas.myurl.net/lbaas/listeners/<listener_id>"

  As a response, it will get containers data like container's resource GET
  request was used.
  Barbican, in its turn will store consumer's (LBaaS instance URL) data
  in its database so this info can be used for getting all consumers of
  a specific TLS container.

  As a result, Neutron LBaaS TLS implementation is required to:

  * Use only POST request to container's consumers resource in order to get
    the container's data.
  * Perform DELETE request to container's consumers resource when
    stop using a container.

* Barbican TLS container will contain PEM encoded data. Specific back-end
  implementation might convert the certificates data to other format
  such as DER, if needed.
* In addition to existing HTTP, HTTPS and TCP, new protocol, TERMINATED_HTTPS
  will be added for listener creation
* For tenant, creating listener with TERMINATED_HTTPS as a protocol means
  desire to offload incoming encrypted traffic. New configuration options
  will be available for listener's configuration including:

    * Default TLS container id for TLS termination
    * TLS containers list for SNI

* In case when specific back-end is not able to support TLS capabilities,
  its driver should throw an exception. The exception message should state
  that this specific back-end (provider) does not support listeners with TLS
  offloading. The clear sign of listener's requirement for TLS
  offloading capabilities is its TERMINATED_HTTPS protocol.

* New module will be developed in Neutron LBaaS for Barbican TLS containers
  interactions. The module will be used by Neutron LBaaS front-end API code
  and providers' driver code. The module will be used for validation and
  data extraction from default TLS container and SNI containers.
  This module represents the only legitimate API for Barbican TLS containers
  interactions. See 'SNI certificates list management' section for detailed
  module specification.

* When creating listener with TERMINATED_HTTPS protocol:

  * Front-end TLS offloading is always enabled - hard coded as a default
    behaviour for listener with TERMINATED_HTTPS protocol
  * Tenant must supply default TLS container for front-end offloading.
    Not supplying a container is an invalid configuration.
  * TLS supported protocols and cipher suites for termination will be
    set to sane values by each back-end's specific code
  * SNI certificates list is optional and not mandatory to specify
  * SNI certificates list specifying is described
    in "SNI certificates list management" section
  * Certificate intermediate chain will be stored as a part of Barbican's
    TLS container
* Back-end re-encryption will not be supported in first phase
* Front-end client authentication and back-end server authentication will not
  be supported in first phase

* When updating listener with TERMINATED_HTTPS protocol:

  * In TLS configuration domain, default TLS container ID for front-end
    offloading and SNI container IDs list are values that may be changed
  * In case when defaut TLS container ID is replaced for the listener,
    back-end implementation should ensure a lack of a downtime on LB appliance.
  * Same for changing SNI container IDs list, back-end should
    avoid LB downtime.

* HA-Proxy LBaaS implementaion and other LBaaS implementations should
  be modified to support this specification.

**With stated above, following is a description of a basic tenant use case - creating listener with TLS offloading:**

* Tenant cteates Barbican TLS container with a certificate.
* Tenant creates listener with TERMINATED_HTTPS as a listener protocol
  and specifies the Barbican TLS container ID as a default TLS container
  for front-end offloading
* As a result, listener created, offloading encrypted traffic on front-end
  with default tenant's TLS certificate, not re-encrypting traffic to the
  back-end.


Requirements from Barbican
--------------------------
* Tenant should be able to create and delete
  TLS containers using Barbican
* Ability to store TLS certificates in Barbican containers
  that contain the TLS certificate itself, its private key
  and optionaly, intermediate chain

  * Creating TLS container with:

    * Certificate : PEM text field
    * Private_key: PEM text_field
    * (extracted) Private_key_pass_phrase : text field
    * Intermediates: PEM text field (optional)
      This field is a concatination of PEM encoded certificate blocks
      in specific order

  * Delete TLS certificate
    **optional:** Check if certificate is in use by any consumer
    and warn before deleting.
    Barbican's BP discussing this feature:
    https://review.openstack.org/#/c/99516/
  * Get TLS container, including private key in PEM encoded PKCS1 or PKCS7
    formats, by container id
  * Get TLS certificate in pem encoded x509 format, by container id


Alternatives
------------

None


Restrictions
------------
* TLS settings are only available for listeners
  having TERMINATED_HTTPS as a protocol. In other cases TLS settings
  will be disabled and have None or empty values.
  There should be a meaningfull error message to a user explaining the exact
  reason of a failure in case of an invalid configuration.
* Listener protocol is immutable. Changing the protocol will require
  radical re-configuration of provider's back-end system, which seems to be
  not  justified for this use case. Tenant should create new listener.
* While updating existing TLS certificate, name and description are only values
  allowed to be modified. Creating new TLS container and using it instead of
  the old one will be easier option than re-configuring LBaaS back-end
  with modified container, at least in first phase.


SNI certificates list management
--------------------------------

For SNI functionality, tenant will supply list of TLS containers in specific
order.
In case when specific back-end is not able to support SNI capabilities,
its driver should throw an exception. The exception message should state
that this specific back-end (provider) does not support SNI capability.
The clear sign of listener's requirement for SNI capability is
a none empty SNI container ids list.
However, reference implementation must support SNI capability.


New separate module will be developed in Neutron LBaaS for Barbican TLS
containers interactions.
The module will use service account for Barbican API interation.
The module will have API for:
* Ensuring Barbican TLS container existence (used by LBaaS front-end API)

* Validating Barbican TLS container (used by LBaaS front-end API)
  This API will also "register" LBaaS as a container's consumer in Barbican's
  repository.

* Extracting SubjectCommonName and SubjectAltName information
  from certificates’ X509 (used by LBaaS front-end API)
  As for now, only dNSName and directoryName types will be extracted from
  SubjectAltName sequence, while directoryName type usage is an issue
  for further discussion.

* Extracting certificate’s data from Barbican TLS container
  (used by provider/driver code)

* Unregistering LBaaS as a consumer of the container when container is not
  used by any listener any more (used by LBaaS front-end API)

The module will use pyOpenSSL and PyASN1 packages.
Only this new common module should be used by Neutron LBaaS code for Barbican
containers interactions.

Front-end LBaaS API (plugin) code will use a new developed module for
validating Barbican TLS containers.
Driver, in its turn, can extract SubjectCommonName and SubjectAltName
information from certificates’ X509 via the common module API
and use it for its specific SNI implementation.

**Note:**

**Specific back-end driver does not have to use SubjectAltName
information. Furthermore, specific driver may throw
an exception saying SubjectAltName is not supported by
its provider**

Any specific driver implementation may extract host names info from
certificates using the mentioned above common module API only, if needed.


**SNI conflicts**

Employing the order of certificates list is not a common requirement
for all back-end implementations.
The order of SNI containers list may be used by specific back-end code,
like Radware's, for specifying priorities among certificates.
Order is meant to be a hint to resolve conflicts when 2 or more certificates
match the DNS name requested in the SNI client hello.
Specific backends might choose to ignore this order and might employ their
own mechanisms to choose one among the clashing certificates.
For ex. NetScaler employs the best match algorithm and does not require
order for conflict resolution.
It's also possible that specific driver throws an exception saying there is
a collision and this specific SNI setup will not be supported by the back-end.


Data model impact
-----------------

**Data model changes**

* *lbaas_listeners* table will be modified with new

  * default_tls_container_id (nullable string 36) - Barbican's
    TLS container id

* New *lbaas_sni* table will be created for storing
  ordered list of TLS containers associated to a listener for SNI capabilities.
  Association objects is composed of:

    * id (immutable string 36) - generated object id
    * listener_id (string 36) - associated listener id
    * tls_container_id (string 36) - associated Barbican TLS container id
    * position - (integer) index for preserving the order


**Required database migration**

* add new columns to *lbaas_isteners* table
* create new *lbaas_sni* table

**New data initial set**

* New columns for *lbaas_listeners* table's existing entries will be set
  to defaults


REST API impact
---------------
**Listener Attributes**

+-------------+-------+---------+---------+------------+----------------------+
|Attribute    |Type   |Access   |Default  |Validation/ |Description           |
|Name         |       |         |Value    |Conversion  |                      |
+=============+=======+=========+=========+============+======================+
|default-tls- |UUID   |RW,tenant|NULL     |UUID        |default TLS cert id   |
|container-id |       |         |         |            |to use for offloading |
+-------------+-------+---------+---------+------------+----------------------+
|sni_container|UUID   |RW,tenant|NULL     |UUID list   |ordered list of       |
|_ids         |list   |         |         |            |TLS containers to use |
|             |       |         |         |            |for SNI              .|
+-------------+-------+---------+---------+------------+----------------------+

**Functions**

* create_listener

  * Creates new listener

  * Request
    *POST /v2.0/lbaas/listeners
    Accept: application/json
    {
    "listener":{
    <...usual listener parameters>,
    "protocol": "TERMINATED_HTTPS"
    "default_tls_container_id": "7804a0de-7f6b-409a-a47c-a1cc7bc77b4j",
    "sni_container_ids": None
    }
    }*

  * Response
    *{
    "listener":{
    "id": "8604a0de-7f6b-409a-a47c-a1cc7bc77b2e"
    <...usual listener parameters>,
    "default_tls_container_id": "7804a0de-7f6b-409a-a47c-a1cc7bc77b4c",
    "sni_container_ids": None
    "tenant_id":"6b96ff0cb17a4b859e1e575d221683d3"
    }
    }*

* create_listener (with SNI list)

  * Creates new listener

  * Request
    *POST /v2.0/lbaas/listeners
    Accept: application/json
    {
    "listener":{
    <...usual listener parameters>,
    "protocol": "TERMINATED_HTTPS"
    "default_tls_container_id": "7804a0de-7f6b-409a-a47c-a1cc7bc77b4j",
    "sni_container_ids": [5404a0de-7f6b-409a-a47c-a1ccgbc77b3j,
    1206a0de-7f6b-409a-a47c-a1ccgbc7bgf3]
    }
    }*

  * Response
    *{
    "listener":{
    "id": "8604a0de-7f6b-409a-a47c-a1cc7bc77b2e"
    <...usual listener parameters>,
    "default_tls_container_id": "7804a0de-7f6b-409a-a47c-a1cc7bc77b4c",
    "sni_container_ids":[5404a0de-7f6b-409a-a47c-a1ccgbc77b3j,
    1206a0de-7f6b-409a-a47c-a1ccgbc7bgf3]
    "tenant_id":"6b96ff0cb17a4b859e1e575d221683d3"
    }
    }*

* update_listener

  * Updates VIP listener

  * Request
    *PUT /v2.0/lbaas/listeners/<listener-id>
    Accept: application/json
    {
    "listener":{
    <...usual listener parameters>,
    "protocol": "TERMINATED_HTTPS"
    "default_tls_container_id": "7804a0de-7f6b-409a-a47c-a1cc7bc77b4c",
    "sni_container_ids": None
    }
    }*

  * Response
    *{
    "listener":{
    "id": "8604a0de-7f6b-409a-a47c-a1cc7bc77b2e"
    <...usual listener parameters>,
    "default_tls_container_id": "7804a0de-7f6b-409a-a47c-a1cc7bc77b4c",
    "sni_container_ids": None
    "tenant_id":"6b96ff0cb17a4b859e1e575d221683d3"
    }
    }*


Security impact
---------------

Following are security requirements:

* Retrieving TLS container from Barbican to LBaaS plugin/driver
  must be secured
* Sending TLS container contents from driver to back-end system
  must be secured
* Storing secrets on neutron server is prohibited
* Back-end systems may need to ensure secured store for secrets
  to meet certain security compliance requirements



Notifications impact
--------------------

None

CLI impact
---------------------

* Listener creation with TERMINATED_HTTPS protocol (default behavior)
  *lb-listener-create* **--protocol TERMINATED_HTTPS**
  *--protocol-port   443*
  **--default_tls_container_id 9a96ff0cb17a4b859e1e575d2216cd23**
  *...<usual CLI options>*

* Listener creation with TERMINATED_HTTPS protocol and SNI certificates list
  *lb-listener-create* **--protocol TERMINATED_HTTPS**
  *--protocol-port 443*
  **--default_tls_container_id 9a96ff0cb17a4b859e1e575d2216cd23
  --sni_container_ids list=true
  6b96ff0cb17a4b859e1e575d221683d3 4596ff0cb17a4b859e1e575d22168ba1**
  *...<usual CLI options>*


Other end user impact
---------------------

None


Performance Impact
------------------

* When updating listener without modifying TLS settings
  (default container id or SNI list) - Barbican API should not be used
  for retrieving container content which was not actually changed.
  This will prevent unnecessary resources consumption when, for example,
  members are added to the pool used by listener.
  It means that each Barbican TLS container will be validated only once
  for a listener while it's still in use by this listener.


Other deployer impact
---------------------

* Barbican is required to be deployed and functional in order this feature
  to work.
* New dependencies are added for neutron, pyOpenSSL and PyASN1. These
  are required by new module for Barbican TLS containers interactions.


Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:

  https://launchpad.net/~evgenyf


Other contributors:
  Barbican TLS containers interactions module -
  https://launchpad.net/~carlos-garza

Work Items
----------

* Develop new module for BArbican TLS containers interactions using pyOpenSSL
  and PyASN1 packages.
* Implement changes in LBaaS DB schema v2
* Implement changes in LBaaS extension v2
* Implement all required CLI changes
* Implement all required unit testing
* Implement all required tempest testing
* Make integration with Barbican certificates storage API
  Detailed specificatio of how Barbican's API for containers should be used
  is at https://review.openstack.org/#/c/99516
* Modifying LBaaS HA-Proxy driver to support TLS capability
  Detailed specification of this work item
  is at https://review.openstack.org/#/c/100931
* Use HA-Proxy version 1.5
* Implement horizon part of this spec, not as part of Juno release.


Dependencies
============

* Barbican API requirements
* Neutron LBaaS API v2 with new listener object implemented
* New dependencies will be added for neutron, pyOpenSSL and PyASN1. These
  are required by new module for Barbican TLS containers interactions.


Testing
=======

**listener unit testing domains**

* REST API and attributes validation tests
* DB mixin and schema tests
* LBaaS Plugin with mocked driver end-to-end tests
* Specific driver tests for each existing driver supporting TLS offloading
* Tempest tests
* CLI tests

Unit testing scenarios

* New listener creation with TERMINATED_HTTPS as a protocol

  * No default TLS container for termination supplied.
    Check error generation
  * Default TLS container for termination supplied.
    Test expected default configuration took place.
  * Default TLS container supplied.
    SNI TLS containers  list was supplied
    Test expected configuration took place

* Update existing listener with TERMINATED_HTTPS as a protocole

  * Change default TLS container. Test expected configuration
  * Add/Modify SNI containers list. Test expected configuration

CLI tests should test inconsistency issues such as:

* No default offloading TLS container specified when creating listener
  with TERMINATES_HTTPS protocole

Documentation Impact
====================

* Neutron API should be modified with new listener TLS attributes
* Neutron CLI should be modified with updated
  listener commands with TLS options


References
==========

* TLS RFC http://tools.ietf.org/html/rfc2818


