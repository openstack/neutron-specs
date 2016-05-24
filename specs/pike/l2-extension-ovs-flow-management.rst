..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
L2 Extension flow management support
====================================

https://bugs.launchpad.net/neutron/+bug/1563967

The l2-agent-extension-api that merged in Mitaka allows extensions of the
Open vSwitch agent to manipulate the Open vSwitch OpenFlow tables.

This introduces a challenge of interoperability between extensions creating
flows.

Problem Description
===================

As the Open vSwitch flow table matches traffic to flows based on packet
characteristics in the order of priority assigned to the flow, it will
inevitably result in two extensions using flows that will block the extension
that uses the lower priority flows.

Therefore a flow manager or flow pipeline is necessary, that would reconcile
flows for multiple extensions in a consistent and mutually compatible way.


Proposed Change
===============

In order to make extensions optional, pluggable and interoperable in the
context of OpenFlow rules we need to define a pipeline model, and a l2
extension API interface, with the main purpose of avoiding cases where
two extensions manipulate the same exact table, inserting eventually
incompatible flows, and to let extensions:

* declare OpenFlowFunctions, where an OpenFlowFunction is a set of tables
  that implement the extension behaviour and can be plugged in either the
  egress or ingress pipeline.

* declare the need to reference other OpenFlowFunctions within the same
  extension (for example an extension could need to learn flows into another
  of its own functions on a different stage of the pipeline ingress/egress,
  a typical example of this is mac-learning ingress function learns to an
  egress function).

* declare OpenFlowRegisters, which will be used inside the function, and
  sometimes can be used by other functions as a medium to exchange metadata.

* declare the need for a specific OpenFlowRegister, generally the needed
  registers are filled or received up by the fixed blocks, but eventually
  extensions, or different extension functions may need to interoperate by
  exchanging metadata in registers.

  Public registers or functions can be accessed by other extensions, but non
  public registers can't. This sets a minimum base for extension interoperation,
  where that makes sense.

Three classes will be provided for the extensions and other OpenFlow mechanisms
(like the OpenFlow firewall driver, or the proposed fixed blocks -see below-)
to declare their OpenFlow functions:

* OpenFlowFunction
* OpenFlowTable
* OpenFlowRegister

The code could roughly look like::

    @openflow_pipeline.register_openflow_function
    class EgressTrafficClassifier(openflow_pipeline.OpenFlowFunction):

        extension = "FIXED_BLOCKS"
        name = 'EGRESS_TRAFFIC_CLASSIFIER'
        bridge = openflow_pipeline.INTEGRATION_BRIDGE

        declares_registers = [OpenFlowRegister(bits=16, name='CLASSIFIER_MARK',
                                               public=True)]
        declares_tables = [OpenFlowTable(name='CLASSIFIER', default=NEXT)]
        needs_registers = [OpenFlowRegister(name='OUTPUT_PORT')]
        needs_function = [OpenFlowfunction(name='INGRESS_TRAFFIC_X'),
                          OpenFlowfunction(name='INPUT_TAGGING')]

        # The openflow manager will register the following attributes on the
        # class once it has resolved all dependencies and creations
        #          <name>_table
        #          <name>_reg
        #          <name>_function
        #
        # for all the declared and needed registers, so in this example
        # we would have the following class attributes:
        #     - classifier_mark_reg
        #     - classifier_table
        #     - output_port_reg
        #     - ingress_traffic_x_function
        #     - input_tagging_function


A top level OpenFlow manager, used by the l2 extension manager will be
responsible of taking all the OpenFlowFunctions declared, resolving order,
and assigning table numbers and registers to every function.

OpenFlowFunctions can choose the bridge or where they want to live
(provider network bridges, integration bridge, tunnel bridge), taking baby
steps we will start by supporting functions that want to live in the
integration bridge, but the bridge intention should be declared in the function
to allow future extensibility.

The OpenFlowManager will inspect the declared tables default action, which
can be:

* NEXT, the lowest priority rule inserted in table jumping to next
  OpenFlow function.
* DROP, the lowest priority rule inserted in the table drops the packet.


Description of the integration bridge pipeline
----------------------------------------------

The pipeline is divided in two paths, from the neutron port of VM instance
point of view:

* Egress path (traffic that leaves the VM/port)
* Ingress path (traffic that goes to a VM/port)


The pipeline looks like this::


                   +----------+
               +<--+ vm-tap-a <---------------+
               |   +----------+               |
               |                              |
               |   +----------+               |
               |<--+ vm-tap-b <-----+         |
               |   +--+-------+     |         |
               |                    |         |
         0+----v------------+ 253+--+---------+--+
          | initial_tagging |  |  input_stage    |
          +-----------+-----+  +--------^--------+
                      |                 |
          +-----------v-----+  +--------+--------+
     E    |  [ function A ] |  | [ function .. ] |
     G    +-----------+-----+  +--------^--------+
     R                |                 |
     E    +-----------v-----+  +--------+--------+
     S    |  [ function ..] |  | [ function F ]  |
     S    +-----------+-----+  +--------^--------+
                      |                 |
     |    +-----------v-----+  +--------+--------+   .
     |    | egress filter   |  |  ingress filter |  /|\
    \|/   +-----------+-----+  +--------^--------+   |
     °                |                 |            |
          +-----------v-----+  +--------+--------+
          |  [ function C ] |  | [ function .. ] |   I
          +-----------+-----+  +--------^--------+   N
                      |                 |            G
          +-----------v-----+  +--------+--------+   R
          |  [ function ..] |  ^  [ function E ] |   E
          +-----------+-----+ /+--------^--------+   S
                      |      /          |            S
          +-----------v-----+ 0+--------+--------+
          |ingress deflector|  | initial_tagging |
          +----------+------+  +---------^-------+
                     |                   |
          +----------v------+            |
          |  output_stage   +------+     |
          +-----------+-----+      |     |
                                   |     |
                                 +-v-----+-------+
                                 | patch-br-xxx  |
                                 +---------------+




Notes:

 * vm-tap-a and vm-tap-b: are VMs or service ports attached to a bridge.
 * patch-br-xxx: is the patch port going to bridge br-xxx
 * Functions can be made of several tables, it's the responsibility of the function
   to jump around its own tables or to the next function once its processing has
   finished.


OpenFlowFunction capabilities
-----------------------------

OpenFlow functions would have the right to add_flows/del_flows to normally
manipulate packets as usual in the next cases:

 * Drop a packet
 * Alter packet details
 * Set a queue
 * push/pop data to the stack (stack size must not change after ending function)
 * learn actions in other functions belonging to the same extension
 * ...
 * etc


Special handling is required for:

 * NORMAL action: normal action should not be executed immediately via action,
   but a register will be marked so the specific packet will be handled with
   "NORMAL" at output stage.
 * Modify destination on the ingress path: the input tagging details must
   be reset, and based on the locality/externality of the packet destination,
   update of input tagging fields (using a helper).
 * Output a clone of the packet:
   "output:X,goto_table:<next-function-first-table>" may be used.
 * Redirect packet to a new destination and stop pipeline processing (for
   extensions that want to move the packet to an ancillary bridge and are
   completely sure that further processing on the bridge should not happen)
   will use output:X with no resubmit to next function.

Helpers will be provided, for cases like the "NORMAL" one, or packet cloning,
and we may want to consider some filter to enforce sanity of the actions.

Notes:

Inserted flows will have to be re-inserted by extensions by calling their
openflow functions as necessary when, for example, openvswitch has been reset
and the flows need to be recreated.

We could have a DSL to define how flows are created for each function, but
that would increase the complexity of this, and we prefer to take baby steps
toward a better integration between extensions. In a future we could consider
such option we have the foundations of a first openflow pipeline.

OpenFlowFunction fixed blocks (integration bridge)
--------------------------------------------------

Some common OpenFlowFunctions are provided by the Open vSwitch agent, the agent
will declare and register those:

* INITIAL_TAGGING: Takes care of identifying the source or destination of the
  packet, marks the network in a register, and sends the packet to the egress
  or ingress path (with priority for egress path when both the source and
  destination of the packet is local)

  Some extensions may want to change the destination, or duplicate packets
  with new destinations. Helpers should be provided for that kind of
  manipulation, so the registers will resemble the same settings done by
  INITIAL_TAGGING function.

  The egress path has priority to cover the case when a packet is both egress
  and ingress in the same bridge (source and destination ports are local
  to the hypervisor)

* INGRESS_DEFLECTOR: For packets that are both egress and ingress on the
  local hypervisor, this stage is in charge of moving
  packets from the end of the egress path, to the start of the ingress path,
  so any ingress filtering or packet processing happens before the packet
  is finally sent to the output port.

* INGRESS_FILTER, EGRESS_FILTER: Is provided by the security groups
  firewall driver (openvswitch firewall), if no OpenFlow filter is provided
  such stage of the pipeline is reserved for the L2 filtering of the hybrid
  firewall used for L2 MAC filtering to overcome the iptables/ebtables
  limitations.

* INPUT_STAGE: Goes after the ingress path, the packet is directed
  to its destination port, or gets the NORMAL rule executed based on the
  packet details. In a future revision, we could get rid of all NORMAL rules,
  with that, transparent vlans and port trunking could be implemented as simple
  OpenFlow functions, with no need for separate bridges.

* OUTPUT_STAGE: Goes after the egress path, the packet is directed to
  its destination patch port or with NORMAL, when going to another bridge.


Each l2-agent extension for the Open vSwitch agent can declare one
or several functional blocks to be plugged.

Every functional block can be composed of one or more OpenFlow tables.

Every functional block has an entry and at least one exit point (next function).
The manager will insert a default jump to next function as the lowest priority
rule for every function table if table "default" is NEXT.


Ordering of functions
---------------------

Ordering of functional blocks is defined in the manager, and maintained in the
core neutron. If somebody wants to declare a new function in the pipeline,
they can come to the community and discuss about its insertion point,
alternatively we will include common extension points that can be used
right away.

COMMON prefixed points can be used by several extensions, and the ordering
between extension functions won't be guaranteed.

Extensions that require several plug points should declare all of them as
separate OpenFlow functions, and make use of them as necessary (note the
<>_<DIR>_PREFILTER / <>_<DIR>_POSTFILTER examples below)

An example of the functional blocks per pipeline could be::

  EGRESS_PIPELINE = [COMMON_EGRESS_PREFILTER0,
                     TAAS_EGRESS_PREFILTER,
                     BGPVPN_EGRESS_PREFILTER,
                     SFC_EGRESS_PREFILTER,
                     COMMON_EGRESS_PREFILTER1,
                     COMMON_EGRESS_FILTER0,
                     EGRESS_FILTER,
                     COMMON_EGRESS_FILTER1,
                     COMMON_EGRESS_POSTFILTER0,
                     QOS_EGRESS_MIN_BW,
                     QOS_EGRESS_DSCP,
                     SFC_EGRESS_POSTFILTER,
                     TAAS_EGRESS_POSTFILTER,
                     COMMON_EGRESS_POSTFILTER1]

  INGRESS_PIPELINE = [COMMON_INGRESS_PREFILTER0,
                      TAAS_INGRESS_PREFILTER,
                      SFC_INGRESS_PREFILTER,
                      COMMON_INGRESS_PREFILTER1,
                      COMMON_INGRESS_FILTER0,
                      INGRESS_FILTER,
                      COMMON_INGRESS_FILTER1,
                      COMMON_INGRESS_POSTFILTER0,
                      TAAS_INGRESS_POSTFILTER,
                      SFC_INGRESS_POSTFILTER,
                      COMMON_INGRESS_POSTFILTER1]

Please note that the fixed blocks (INITIAL_TAGGING, INPUT_STAGE, OUTPUT_STAGE,
INGRESS_DEFLECTOR) are not included in the configuration definition, as are
neither ingress or egress pipeline, but blocks that will steer traffic to the
right pipelines.

References
==========

[1] https://www.opennetworking.org/images/stories/downloads/sdn-resources/onf-specifications/openflow/openflow-switch-v1.4.1.pdf

