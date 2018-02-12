
OpenFlow in POX
---------------

One of the primary purposes for using POX is for developing OpenFlow control applications – that is, where POX acts as a controller for an OpenFlow switch (or, in more proper terminology, an OpenFlow *datapath*).  In this chapter, we describe some of the POX features and interfaces that facilitate this, beginning with a quick overview of some of the major pieces.

Because POX is so often used with OpenFlow, there is a special demand-loading mechanism, which will usually detect when you're trying to use OpenFlow, and load up OpenFlow-related components with default values.  See the "About the OpenFlow Component's Initialization" subsection for more information on this.  If the demand loading doesn't detect that you're trying to use it, you can either tweak your component to make it clear that you are (simply accessing core.openflow in your launch function should do it), or simply specify the ":mono:`openflow`" component at the start of the commandline.

A main part of the POX OpenFlow API is the OpenFlow "nexus" object.  Usually, there is a single such object which is registered as :mono:`core.openflow` as part of the demand-loading process mentioned above.  Some usage of this nexus object is explored in following subsections, including one subsection dedicated entirely to it.

The POX component that actually communicates with OpenFlow switches is :mono:`openflow.of_01` (the 01 refers to the fact that this component speaks OpenFlow wire protocol 0x01). Again, the demand-loading feature will usually cause this component to be initialized with default values (listening on port 6633).  However, you can invoke it automatically instead to either change the options, or because you want to run it multiple times (e.g., to listen on plain TCP and SSL or on multiple ports).  See the documentation for the :mono:`of_01` component for further details.


DPIDs in POX
============

Before we truly begin discussing the details of communicating with OpenFlow datapath, we should discuss the subject of DPIDs.  The OpenFlow specification specifies that datapaths (switches) each have a unique datapath ID or DPID, which is a 64 bit value, and is communicated from the switch to the controller during handshaking by way of the :mono:`ofp_switch_features` message.  It puts forth that 48 of those bits are intended to be an Ethernet address and that 16 are "implementer-defined" (in practice, they are very often just zero).  Since an OpenFlow switch is itself (mostly) "transparent" to the network, it's not entirely clear exactly *which* Ethernet address is supposed to be in those bits, but we can assume it's something switch-specific.  Since OpenFlow Connection objects (discussed below) are tied to a specific switch, the DPID is available on the Connection object using the :mono:`.dpid` attribute.  Additionally, the corresponding Ethernet address is available using the :mono:`.eth_addr` attribute.

POX internally treats the DPIDs as Python integer types.  This isn't that nice for humans, though.  If you print them out they're just a decimal number which may not be easy to look at or easy to correlated with the associated Ethernet address.  Therefore, POX defines a specific way of formatting DPIDs, which is implemented in :mono:`pox.lib.util.dpid_to_str()`.  When passed a DPID in the common case that the 16 "implementer-defined" bits are zeros, the result is a string which looks very much like an Ethernet address except that instead of colons separating the bytes (as POX always does for Ethernet addresses), dashes are used instead.  If the implementer-defined bits are nonzero, they are treated as a decimal number and appended following a bar, e.g., "00-00-00-00-00-05|123".  The second parameter of :mono:`dpid_to_str()` allows you to force that the long format always be used.  That is, it defaults to :mono:`False`, but if you pass in :mono:`True`, the 16 extra bits are shown even when they're zero.  There is also a corresponding :mono:`str_to_dpid()` function which attempts to parse strings as DPIDs (returning an integer/long).


DPIDs in Mininet
****************

Although this isn't specific to POX, it is worth saying a few words about DPIDs in Mininet.  By default, Mininet assigns DPIDs to switches in a straighforward way.  If a switch is "s3", then its DPID will be 3.  This can be problematic when used with the :mono:`--mac` option.  The :mono:`--mac` option assigns MAC addresses to hosts in much the same way – if a host is "h3" then its MAC will be 00:00:00:00:00:03.  While this can be helpful, it also means that the portion of the DPID which the OpenFlow specification says is intended to be a MAC address is the same as the MAC address of one of the hosts.  This can be a source of confusion and problems since MACs are generally assumed to be unique.

Some POX components make a particular OpenFlow switch act like something besides a transparent L2 switch.  For example, :mono:`arp_responder` makes an OpenFlow switch act a tiny bit more like a router.  Routers have Ethernet addresses, so... which Ethernet address should :mono:`arp_responder` use?  There are lots of answers here, but one reasonable one is to use the one that's embedded in the DPID (and available on the :mono:`Connection`'s, :mono:`.eth_addr` attribute).  As you can see, this has the potential to cause address conflicts when using Mininet's :mono:`--mac` option.  There are ways around this type of situation, but it's helpful to be aware of the issue.


Communicating with Datapaths (Switches)
=======================================

Switches connect to POX, and then you obviously want to communicate with those switches from POX.  This communication might go either from the controller to a switch, or from a switch to the controller.  When communication is from the controller to the switch, this is performed by controller code which sends an OpenFlow message to a particular switch (more on this in a moment).  When messages are coming from the switch, they show up in POX as *events* for which you can write event handlers – generally there's an event type corresponding to each message type that a switch might send.  While the messages themselves are described in the OpenFlow specification and the events are described in following subsections, this subsection focuses simply on how exactly you send those messages and how you set up those event handlers.

There are essentially two ways you can communicate with a datapath in POX: via a :mono:`Connection` object for that particular datapath or via an OpenFlow Nexus which is managing that datapath.  There is one :mono:`Connection` object for each datapath connected to POX, and there is typically one OpenFlow Nexus that manages all connections.  In the normal configuration, there is a single OpenFlow nexus which is available as :mono:`core.openflow`.  There is a lot of overlap between :mono:`Connections` and the Nexus.  Either one can be used to send a message to a switch, and most events are raised on both.  Sometimes it's more convenient to use one or the other.  If your application is interested in events from all switches, it may make sense to listen to the Nexus, which raises events for all switches.  If you're interested only in a single switch, it may make sense to listen to the specific Connection.


Connection Objects
******************

Every time a switch connects to POX, there is also an associated :mono:`Connection` object.  If your code has a reference to that :mono:`Connection` object, you can use its :mono:`send()` method to send messages to the datapath.

:mono:`Connection` objects, along with being able to send commands to switches and being sources of events from switches, have a number of other useful attributes.  We list some here (for more, view the reference for the :mono:`Connection` class):

================= ========================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================
member            description
================= ========================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================
:mono:`ofnexus`   A reference to the nexus object associated with this connection.  (Usually this is the same as core.openflow.)
:mono:`dpid`      The datapath identifier of the switch. (See the next section for more details.)
:mono:`features`  The switch features reply (ofp_switch_features) sent by the switch during handshaking.
:mono:`ports`     The ports on the switch.  As these may change during the lifetime of a connection, POX *attempts* to track such changes.  However, there is always the possibility that these are out of date (hopefully only transiently). |brk| This attribute is a reference to a special :mono:`PortCollection` object.  This object is sort of like a dictionary where values are :mono:`ofp_phy_port` objects and the keys are flexible – you can look up ports by their OpenFlow port number (:mono:`ofp_phy_port`'s :mono:`.port_no`), their Ethernet address (:mono:`ofp_phy_port`'s :mono:`.hw_addr`), or their port name (:mono:`ofp_phy_port`'s :mono:`.name`, e.g., "eth0").
:mono:`sock`      The socket connecting to the peer.  This is a Python socket object, so you can, e.g., retrieve the address of the switch's side of the connection using :mono:`connection.sock.getpeername()`.
:mono:`send(msg)` A method used to send an OpenFlow message to the switch.
================= ========================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================



In addition to its attributes and the send() method, Connection objects raise events corresponding to particular datapaths, for example when a datapath disconnects or sends a notification (for more on events in general, see the section "The Event System").  You can create handlers for events on a particular datapath by registering event listeners on the associated Connection.  You can find examples of this later in this section.


Getting a Reference to a Connection Object
##########################################

If you wish to use any of the above-mentioned attributes of a :mono:`Connection` object, you – of course – need a reference to the :mono:`Connection` object associated with the datapath you're interested in.  There are three major ways to get such a reference to a :mono:`Connection` object:

#. You can listen to :mono:`ConnectionUp` events on the nexus – these pass the new :mono:`Connection` object along
#. You can use the nexus's :mono:`getConnection(<DPID>)` method to find a connection by the switch's DPID (see the next section)
#. You can enumerate all of the nexus's connections via its :mono:`connections` property (e.g., :mono:`for con in core.openflow.connections`) (see the next section)

As an example of the first, you may have code in your own component class which tracks connections and stores references to them itself.  It does this by listening to the :mono:`ConnectionUp` event on the OpenFlow nexus.  This event includes a reference to the new connection, which is added to its own set of connections.  The following code demonstrates this (note that a more complete implementation would also want to use the :mono:`ConnectionDown` event to remove :mono:`Connection`\s from the set!).


.. code-block:: python

  class MyComponent (object):
      def __init__ (self):
          self.connections = set()
          core.openflow.addListeners(self)

      def _handle_ConnectionUp (self, event):
          self.connections.add(event.connection) # See ConnectionUp event documentation



The OpenFlow Nexus – core.openflow
**********************************

An OpenFlow nexus is essentially a manager for a set of OpenFlow :mono:`Connections`.  Typically, there is a single nexus which manages connections to all switches, and this is available as :mono:`core.openflow`.  (The advanced topic of creating multiple nexus objects and assigning particular connections to each one via a connection arbiter object is an advanced topic for very particular use cases and is not currently covered in this manual.)

Here we list some attributes of a nexus:

======================================== ========================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================
attribute                                description
======================================== ========================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================
:mono:`miss_send_len`                    When a packet does not match any table entry on a datapath, the datapath will forward the packet to the controller inside a packet-in message.  To conserve bandwidth, the datapath will actually not send the entire packet, but only the first :mono:`miss_send_len` bytes.  By adjusting this value here, any datapaths which subsequently connect will be configured to only send this number of bytes. |brk|   This defaults to OFP_DEFAULT_MISS_SEND_LEN from the OpenFlow specification (128 bytes).
:mono:`clear_flows_of_connect`           When True (the default), POX will delete all flows on the first table of a switch when it connects.
:mono:`connections`                      A special collection (see below) containing references to all connections this nexus is handling.
:mono:`getConnection(<dpid>)`            Get a connection object for a particular datapath via its DPID or None if not available.
:mono:`sendToDPID(<dpid>,<msg>)`         Send an OpenFlow message to a particular datapath, dropping the message (and logging a warning) if the datapath isn't connected.  (:mono:`Similar to doing core.openflow.getConnection(dpid).send(msg)`).
======================================== ========================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================

The connections collection is essentially a dictionary where the keys are DPIDs and the values are Connection objects.  However, if you iterate this, it iterates the Connections and not the DPIDs, unlike a normal dictionary.  To iterate the DPIDs, you can use the :mono:`.iter_dpids()` method. Additionally, you can use the "in" operator to check for whether a :mono:`Connection` is in this collection as well as whether a DPID is in the collection, and there is a :mono:`.dpids` attribute as a more usecase-specific alternative to the generic :mono:`.keys()`.

As with Connection objects, you can also set event listeners on the nexus object itself.  Whereas a Connection object only raises events pertaining to the datapath associated with that particular Connection, the nexus object raises events relevant to *any* of the Connections it's managing.  We dig in to these events in the next subsection.

.. todo:: Add notes on order of events between nexus and Connection, halting events, etc.


OpenFlow Events: Responding to Switches
=======================================

_Note: For more background on the event system in POX, see the relevant section in this manual._

Most OpenFlow related events are raised in direct response to a message received from a switch.  As a general guideline, OpenFlow related events have the following three attributes:

========== =================== ====================================================================================================
attribute  type                description
========== =================== ====================================================================================================
connection Connection          Connection to the relevant switch (e.g., which sent the message this event corresponds to).
dpid       long                Datapath ID of relevant switch (use dpid_to_str() to format it for display).
ofp        ofp_header subclass OpenFlow message object that caused this event.  See `OpenFlow Messages`_ for info on these objects.
========== =================== ====================================================================================================




In the rest of this section, we describe some of the events provided by the OpenFlow module and topology module.  To get you started, here's a very simple POX component that listens to :mono:`ConnectionUp` events from all switches, and logs a message when one occurs.  You can put this into a file (e.g., :mono:`ext/connection_watcher.py`) and then run it (with :mono:`./pox.py connection_watcher`) and watch switches connect.


.. code-block:: python

  from pox.core import core
  from pox.lib.util import dpid_to_str

  log = core.getLogger()

  class MyComponent (object):
    def __init__ (self):
      core.openflow.addListeners(self)

    def _handle_ConnectionUp (self, event):
      log.debug("Switch %s has come up.", dpid_to_str(event.dpid))

  def launch ():
    core.registerNew(MyComponent)



ConnectionUp
************

Unlike most other OpenFlow events, this message is not raised in response to reception of a specific OpenFlow message from a switch – it's simply fired in response to the establishment of a new control channel with a switch.

Also note that while most OpenFlow events are raised on both the Connection itself and on the OpenFlow nexus, the :mono:`ConnectionUp` event is raised only on the nexus.  This makes sense since the :mono:`ConnectionUp` event is the first sign that a :mono:`Connection` exists – nobody could have possibly set a listener on it yet!

Additional attribute information (in addition to the standard OpenFlow event attributes):

========= =================== =======================================================================================================================================================================================================================================================
attribute type                notes
========= =================== =======================================================================================================================================================================================================================================================
ofp       ofp_switch_features Contains information about the switch, for example supported action types (e.g., whether field rewriting is available), and port information (e.g., MAC addresses and names).  (This is also available on the Connection's :mono:`features` attribute.)
========= =================== =======================================================================================================================================================================================================================================================

This event can be handled as shown below:


.. code-block:: python

  def _handle_ConnectionUp (self, event):
    print "Switch %s has come up." % event.dpid



ConnectionDown
**************

Similar to :mono:`ConnectionUp` but unlike most other OpenFlow-related events, this event is not fired in response to an actual OpenFlow message.  It is simply fired when a connection to a switch has been terminated (either because it has been closed explicitly, because the switch was restarted, etc.).

Note that unlike :mono:`ConnectionUp`, this event is raised on both the nexus and the :mono:`Connection` itself.

Note that this event has no :mono:`.ofp` attribute.


PortStatus
**********

:mono:`PortStatus` events are raised when the controller receives an OpenFlow port-status message (:mono:`ofp_port_status`) from a switch, which indicates that ports have changed.  Thus, its :mono:`.ofp` attribute is an :mono:`ofp_port_status`.


.. code-block:: python

  class PortStatus (Event):
    def __init__ (self, connection, ofp):
      Event.__init__(self)
      self.connection = connection
      self.dpid = connection.dpid
      self.ofp = ofp
      self.modified = ofp.reason == of.OFPPR_MODIFY
      self.added = ofp.reason == of.OFPPR_ADD
      self.deleted = ofp.reason == of.OFPPR_DELETE
      self.port = ofp.desc.port_no



A quick example:


.. code-block:: python

  def _handle_PortStatus (self, event):
    if event.added:
      action = "added"
    elif event.deleted:
      action = "removed"
    else:
      action = "modified"
    print "Port %s on Switch %s has been %s." % (event.port, event.dpid, action)



FlowRemoved
***********

:mono:`FlowRemoved` events are raised when the controller receives an OpenFlow flow-removed message (:mono:`ofp_flow_removed`\) from a switch, which are sent when a table entry is removed on the switch either due to a timeout or explicit deletion.  Such notifications are sent only when the flow was installed with the :mono:`OFPFF_SEND_FLOW_REM` flag set.  See the OpenFlow specification for further details.

While you can, as usual, access the :mono:`ofp_flow_removed` directly via the event's :mono:`.ofp` attribute, the event has several attributes for convenience:

=========== ==== ===============================================
attribute   type meaning
=========== ==== ===============================================
idleTimeout bool True if entry was removed due to idleness
hardTimeout bool True if entry was removed due to a hard timeout
timeout     bool True if entry was removed due to any timeout
deleted     bool True if entry was explicitly deleted
=========== ==== ===============================================


Statistics Events
*****************

Statistics events are raised when the controller receives an OpenFlow statistics reply message (:mono:`ofp_stats_reply / OFPT_STATS_REPLY`) from a switch, which is sent in response to a statistics request sent by the controller.

There are a number of statistics events.  The most basic is :mono:`RawStatsReply` which is simply fired in response to an ofp_stats_reply message from the switch.  However, this message (and therefore the associated event) is not particularly convenient, as it's up to the user to determine what type of statistics event it is, and possibly to "glue back together" multi-part statistics replies.

To remedy this, POX includes separate events for each statistics reply type, and these events are fired when the entire response (including possible multiple parts) have been received.  If none of this makes any sense to you because you haven't read the OpenFlow specification thoroughly – that's fine.  The short of it is that you should just handle the event for the specific stats type that you're interested in.  These include:




================================== =================================
Event                              OpenFlow Stats Type
================================== =================================
:mono:`SwitchDescReceived`         :mono:`ofp_desc_stats`
:mono:`FlowStatsReceived`          :mono:`ofp_flow_stats`
:mono:`AggregateFlowStatsReceived` :mono:`ofp_aggregate_stats_reply`
:mono:`TableStatsReceived`         :mono:`ofp_table_stats`
:mono:`PortStatsReceived`          :mono:`ofp_port_stats`
:mono:`QueueStatsReceived`         :mono:`ofp_queue_stats`
================================== =================================

Underneath, each of these events is a subclass of the :mono:`StatsReply` superclass.  When handling these :mono:`StatsReply`-based events, the :mono:`.stats` attribute will contain a complete set of statistics (e.g., an array of :mono:`ofp_flow_stats `for:mono:` FlowStatsReceived`).  See the section on :mono:`ofp_stats_request` for more information.  More specifically, note the following information for all :mono:`StatsReply` subclasses:

============= =====================================================================================================================================================================================================================================
attribute     meaning
============= =====================================================================================================================================================================================================================================
:mono:`ofp`   Because a :mono:`StatsReply` may have glued together multiple individual OpenFlow messages, the :mono:`.ofp` attribute is a _list_ of :mono:`ofp_stats_reply` messages.  (In the typical case, however, the list has a single entry.)
:mono:`stats` All of the individual stats bodies in a single list.
============= =====================================================================================================================================================================================================================================


PacketIn
********

Fired when the controller receives an OpenFlow packet-in message (:mono:`ofp_packet_in / OFPT_PACKET_IN`) from a switch, which indicates that a packet arriving at a switch port has either failed to match all entries in the table, or the matching entry included an action specifying to send the packet to the controller.

In addition to the usual OpenFlow event attributes:

* port (int) - number of port the packet came in on 
* data (bytes) - raw packet data 
* parsed (packet subclasses) - pox.lib.packet's parsed version
* ofp (ofp_packet_in) - OpenFlow message which caused this event

ErrorIn
*******

Fired when the controller receives an OpenFlow error (:mono:`ofp_error_msg / OFPT_ERROR_MSG`) from a switch.

In addition to the usual OpenFlow event attributes:

========== ==============================================================================================================================================================
attribute  meaning
========== ==============================================================================================================================================================
should_log Usually, an OpenFlow error results in a log message.  If you handle the ErrorIn event, you may set this attribute to False to silence the default log message.
asString() Formats this error as a string.
========== ==============================================================================================================================================================





BarrierIn
*********




Fired when the controller receives an OpenFlow barrier reply (:mono:`OFPT_`:mono:`BARRIER_R`:mono:`EPLY`) from a switch, which indicates that the switch has finished processing commands sent by the controller prior to the corresponding barrier request.

In addition to the usual attributes for OpenFlow events, the BarrierIn event contains:

========= ======= =========================================================================================================================================================================================================================================================================================
attribute type    description
========= ======= =========================================================================================================================================================================================================================================================================================
xid       integer Transaction ID.  For events which are responses to commands sent by the controller, this will contain the same value as the :mono:`.xid` of the command.  For instance, a :mono:`BarrierIn`'s :mono:`.xid` will be the same value as was used in the :mono:`ofp_barrier_request` message.
========= ======= =========================================================================================================================================================================================================================================================================================


OpenFlow Messages
=================

OpenFlow messages are how OpenFlow switches communicate with controllers.  The messages are defined in the OpenFlow Specification.  There are multiple versions of the specification; POX currently supports OpenFlow version 1.0.0 (wire protocol version 0x01).

POX contains classes and constants corresponding to elements of the OpenFlow protocol, and these are defined in the file :mono:`pox/openflow/libopenflow_01.py` (the 01 referring to the wire protocol version). For the most part, the names are the same as they are in the specification.  In a few instances, POX has names which we think are better.  Additionally, POX defines some classes do not correspond to specific structures in the specification (the specification does not describe structs which are just a plain OpenFlow header only differentiated by the message type attribute – POX does).  Thus, you may well wish to refer to the `OpenFlow Specification <http://www.openflow.org/documents/openflow-spec-v1.0.0.pdf>`_ itself in addition to this document (and, of course, the POX code and pydoc/Sphinx reference).

A nice aspect of POX's OpenFlow library is that many fields have useful default values or can infer values.

In the following subsections, we will discuss a useful subset of POX's OpenFlow interface.

.. todo:: Redo following sections to have tables of values/types/descriptions rather than snippets from init functions.


ofp_packet_out - Sending packets from the switch
************************************************

The main purpose of this message is to instruct a switch to send a packet (or enqueue it).  However it can also be useful as a way to instruct a switch to discard a buffered packet (by simply not specifying any actions).

========= ================================ ========= ==================================================================================================================================================================================================================================================================
attribute type                             default   notes
========= ================================ ========= ==================================================================================================================================================================================================================================================================
buffer_id int/None                         None      ID of the buffer in which the packet is stored at the datapath. If you're not resending a buffer by ID, use None.
in_port   int                              OFPP_NONE Switch port that the packet arrived on if resending a packet.
actions   list of ofp_action_XXXX          [ ]       If you have a single item, you can also specify this using the named parameter "action" of the initializer.
data      bytes / ethernet / ofp_packet_in ''        The data to be sent (or None if sending an existing buffer via its buffer_id). |brk| If you specify an :mono:`ofp_packet_in` for this, :mono:`in_port`, :mono:`buffer_id`, and :mono:`data` will all be set correctly – this is the easiest way to resend a packet.
========= ================================ ========= ==================================================================================================================================================================================================================================================================


.. note:: If you receive an :mono:`ofp_packet_in` and wish to resend it, you can simply use it as the :mono:`data` attribute.

See section of 5.3.6 of OpenFlow 1.0 spec. This class is defined in pox/openflow/libopenflow_01.py.


ofp_flow_mod - Flow table modification
**************************************



.. code-block:: python

  class ofp_flow_mod (ofp_header):
    def __init__ (self, **kw):
      ofp_header.__init__(self)
      self.header_type = OFPT_FLOW_MOD
      if 'match' in kw:
        self.match = None
      else:
        self.match = ofp_match()
      self.cookie = 0
      self.command = OFPFC_ADD
      self.idle_timeout = OFP_FLOW_PERMANENT
      self.hard_timeout = OFP_FLOW_PERMANENT
      self.priority = OFP_DEFAULT_PRIORITY
      self.buffer_id = None
      self.out_port = OFPP_NONE
      self.flags = 0
      self.actions = []



* cookie (int) - identifier for this flow rule. (optional)
* command (int) - One of the following values:

 * OFPFC_ADD - add a rule to the datapath (default)
 * OFPFC_MODIFY - modify any matching rules
 * OFPFC_MODIFY_STRICT - modify rules which strictly match wildcard values.
 * OFPFC_DELETE - delete any matching rules
 * OFPFC_DELETE_STRICT - delete rules which strictly match wildcard values.

* idle_timeout (int) - rule will expire if it is not matched in 'idle_timeout' seconds. A value of OFP_FLOW_PERMANENT means there is no idle_timeout (the default).
* hard_timeout (int) - rule will expire after 'hard_timeout' seconds. A value of OFP_FLOW_PERMANENT means it will never expire (the default)
* priority (int) - the priority at which a rule will match, higher numbers higher priority. Note: Exact matches will have highest priority.
* buffer_id (int) - A buffer on the datapath that the new flow will be applied to.  Use None for none.  Not meaningful for flow deletion.
* out_port (int) - This field is used to match for DELETE commands.OFPP_NONE may be used to indicate that there is no restriction.
* flags (int) - Integer bitfield in which the following flag bits may be set:

 * OFPFF_SEND_FLOW_REM - Send flow removed message to the controller when rule expires
 * OFPFF_CHECK_OVERLAP - Check for overlapping entries when installing. If one exists, then an error is send to controller
 * OFPFF_EMERG - Consider this flow as an emergency flow and only use it when the switch controller connection is down.

* actions (list) - actions are defined below, each desired action object is then appended to this list and they are executed in order.
* match (ofp_match) - the match structure for the rule to match on (see below).

See section of 5.3.3 of OpenFlow 1.0 spec. This class is defined in pox/openflow/libopenflow_01.py.


Example: Installing a table entry
#################################



.. code-block:: python

  # Traffic to 192.168.101.101:80 should be sent out switch port 4

  # One thing at a time...
  msg = of.ofp_flow_mod()
  msg.priority = 42
  msg.match.dl_type = 0x800
  msg.match.nw_dst = IPAddr("192.168.101.101")
  msg.match.tp_dst = 80
  msg.actions.append(of.ofp_action_output(port = 4))
  self.connection.send(msg)

  # Same exact thing, but in a single line...
  self.connection.send( of.ofp_flow_mod( action=of.ofp_action_output( port=4 ),
                                         priority=42,
                                         match=of.ofp_match( dl_type=0x800,
                                                             nw_dst="192.168.101.101",
                                                             tp_dst=80 )))



Example: Clearing tables on all switches
########################################



.. code-block:: python

  # create ofp_flow_mod message to delete all flows
  # (note that flow_mods match all flows by default)
  msg = of.ofp_flow_mod(command=of.OFPFC_DELETE)

  # iterate over all connected switches and delete all their flows
  for connection in core.openflow.connections: # _connections.values() before betta
    connection.send(msg)
    log.debug("Clearing all flows from %s." % (dpidToStr(connection.dpid),))



ofp_stats_request - Requesting statistics from switches
*******************************************************



.. code-block:: python

  class ofp_stats_request (ofp_header):
    def __init__ (self, **kw):
      ofp_header.__init__(self)
      self.header_type = OFPT_STATS_REQUEST
      self.type = None # Try to guess
      self.flags = 0
      self.body = b''


* type (int) - The type of stats request (e.g., OFPST_PORT).  Default is to try to guess based on *body*.
* flags (int) - No flags are defined in OpenFlow 1.0.
* body (flexible) - The body of the stats request.  This can be a raw bytes object, or a packable class (e.g., ofp_port_stats_request).

See section of 5.3.5 of OpenFlow 1.0 spec for more info on this structure and on the individual statistics types (port stats, flow stats, aggregate flow stats, table stats, etc.). This class is defined in pox/openflow/libopenflow_01.py

.. todo:: Show some of the individual stats request/reply types?


Example - Web Flow Statistics
#############################

Request the flow table from a switch and dump info about web traffic.  This example is meant to be run along with, say, the forwarding.l2_learning component.  It can be pasted into the POX interactive interpreter (if you run POX including the :mono:`py` component).  There is also an extended version of this example meant to run as a component in the Third Party section – the "Statistics Collector Example".

See the Statistics Events section for more info.


.. code-block:: python
  :linenos:

  import pox.openflow.libopenflow_01 as of
  log = core.getLogger("WebStats")

  # When we get flow stats, print stuff out
  def handle_flow_stats (event):
    web_bytes = 0
    web_flows = 0
    for f in event.stats:
      if f.match.tp_dst == 80 or f.match.tp_src == 80:
        web_bytes += f.byte_count
        web_flows += 1
    log.info("Web traffic: %s bytes over %s flows", web_bytes, web_flows)

  # Listen for flow stats
  core.openflow.addListenerByName("FlowStatsReceived", handle_flow_stats)

  # Now actually request flow stats from all switches
  for con in core.openflow.connections: # make this _connections.keys() for pre-betta
    con.send(of.ofp_stats_request(body=of.ofp_flow_stats_request()))



Match Structure
===============

OpenFlow defines a match structure – :mono:`ofp_match` – which enables you to define a set of headers for packets to match against. You can either build a match from scratch, or use a factory method to create one based on an existing packet.

The match structure is defined in pox/openflow/libopenflow_01.py in class :mono:`ofp_match`.  Its attributes are derived from the members listed in the OpenFlow specification, so refer to that for more information, though they are summarized in the table below.

:mono:`ofp_match` attributes:

=========== =========================================================
Attribute   Meaning
=========== =========================================================
in_port     Switch port number the packet arrived on
dl_src      Ethernet source address
dl_dst      Ethernet destination address
dl_vlan     VLAN ID
dl_vlan_pcp VLAN priority
dl_type     Ethertype / length (e.g. 0x0800 = IPv4)
nw_tos      IP TOS/DS bits
nw_proto    IP protocol (e.g., 6 = TCP) or lower 8 bits of ARP opcode
nw_src      IP source address
nw_dst      IP destination address
tp_src      TCP/UDP source port
tp_dst      TCP/UDP destination port
=========== =========================================================

Attributes may be specified either on a match object or during its initialization.  That is, the following are equivalent:


.. code-block:: python

  my_match = of.ofp_match(in_port = 5, dl_dst = EthAddr("01:02:03:04:05:06"))
  #.. or ..
  my_match = of.ofp_match()
  my_match.in_port = 5
  my_match.dl_dst = EthAddr("01:02:03:04:05:06")



Partial Matches and Wildcards
*****************************

Unspecified fields are *wildcarded* and will match any packet.  You can explicitly set a field to be wildcarded by setting it to :mono:`None`.

.. note:: *Info:* While the OpenFlow :mono:`ofp_match` structure is defined as having a :mono:`wildcards` attribute, *you will probably never need to explicitly set it when using POX* -- simply don't assign values to fields you want wildcarded (or set them to :mono:`None`).

IP address fields are a bit trickier, as they can be wildcarded completely like the other fields, but can also be *partially* wildcarded.  This allows you to match entire subnets.  There are a number of ways to do this.  Here are some equivalent ones:


.. code-block:: python

  my_match.nw_src = "192.168.42.0/24"
  my_match.nw_src = (IPAddr("192.168.42.0"), 24)
  my_match.nw_src = "192.168.42.0/255.255.255.0"
  my_match.set_nw_src(IPAddr("192.168.42.0"), 24)


In particular, note that the :mono:`nw_src` and :mono:`nw_dst` attributes can be ambiguous when working with partial matches – especially when reading a match structure (e.g., as returned in a flow_removed message or flow_stats reply).  To account for this, you may use the unambiguous :mono:`.get_nw_src()`, :mono:`.set_nw_src()`, and the destination equivalents.  These return a tuple such as :mono:`(IPAddr("192.168.42.0"), 24)` which includes the number of matched bits – the number that would follow the slash in CIDR-style representation (192.168.42.0/24).

Note that some fields have *prerequisites*.  Basically this means that you can't specify higher-layer fields without specifying the corresponding lower-layer fields also.  For example, you can not create a match on a TCP port without also specifying that you wish to match TCP traffic.  And in order to match TCP traffic, you must specify that you wish to match IP traffic.  Thus, a match with only :mono:`tp_dst=80`, for example, is invalid.  You must also specify :mono:`nw_proto=6` (TCP), and :mono:`dl_type=0x800` (IPv4).  If you violate this, you should get the warning message ':mono:`Fields ignored due to unspecified prerequisites`'.  For more information on this subject, see the FAQ entry "I tried to install a table entry but got a different one.  Why?".


ofp_match Methods
*****************


======================================================= ================================================================================================================================================================================================
Method                                                  Description
======================================================= ================================================================================================================================================================================================
from_packet(*packet, in_port=None, spec_frags=False*)   Class factory.  See "Defining a match from an existing packet" below.
clone()                                                 Returns a copy of this ofp_match.
flip()                                                  Returns a copy with its source and destinations reversed.
show()                                                  Returns a large string representation.
get_nw_src()                                            Returns the IP source address and the number of matched bits as a tuple.  For example: (IPAddr("192.168.42.0", 24).  Note that the first element of the tuple will be None when the second is 0.
set_nw_src(IP and bits)                                 Sets the IP source address and the number of bits to match.  The arguments can either be two arguments (one for IP and one for bit count), or a tuple in the format used by get_nw_src().
get_nw_dst()                                            Same as get_nw_src() but for destination address.
set_nw_dst(IP and bits)                                 Same as set_nw_src() but for destination address.
======================================================= ================================================================================================================================================================================================


Defining a match from an existing packet
****************************************

There is a simple way to create an exact match based on an existing packet object (that is, an :mono:`ethernet` object from :mono:`pox.lib.packet`) or from an existing :mono:`ofp_packet_in`.  This is done using the factory method :mono:`ofp_match.from_packet()`.


.. code-block:: python

  my_match = ofp_match.from_packet(packet, in_port)


The :mono:`packet` parameter is a parsed packet or :mono:`ofp_packet_in` from which to create the match.  As the input port is not actually in a packet header, the resulting match will have the input port wildcarded by default when this method is called with a packet.  You can, of course, set the in_port field later yourself, but as a shortcut, you can simply pass it in to from_packet().  When using from_packet() with an ofp_packet_in, the in_port is taken from there by default.

Note that you can set fields of the resultant match object to None (wildcarding them) if you want a less-than-exact match.

from_packet() also has an optional spec_frags argument which defaults to False.  See page 9 of the OpenFlow 1.0 specification to help understand the rationale for its existence.


Example: Matching Web Traffic
*****************************

As an example, the following code will create a match for traffic to web servers:


.. code-block:: python

  import pox.openflow.libopenflow_01 as of # POX convention
  import pox.lib.packet as pkt # POX convention
  my_match = of.ofp_match(dl_type = pkt.ethernet.IP_TYPE, nw_proto = pkt.ipv4.TCP_PROTOCOL, tp_dst = 80)



OpenFlow Actions
================

OpenFlow actions are applied to packets that match a rule installed at the datapath. The code snippets found here can be found in libopenflow_01.py in pox/openflow.


Output
******

Forward packets out of a physical or virtual port. Physical ports are referenced to by their integral value, while virtual ports have symbolic names. Physical ports should have port numbers less than 0xFF00.

Structure definition:


.. code-block:: python

  class ofp_action_output (object):
    def __init__ (self, **kw):
      self.port = None # Purposely bad -- require specification


* port (int) the output port for this packet. Value could be an actual port number or one of the following virtual ports:

 * OFPP_IN_PORT - Send back out the port the packet was received on.  Except possibly OFPP_NORMAL, *this is the only way to send a packet back out its incoming port.*
 * OFPP_TABLE - Perform actions specified in flowtable. Note: Only applies to ofp_packet_out messages.
 * OFPP_NORMAL - Process via normal L2/L3 legacy switch configuration (if available – switch dependent)
 * OFPP_FLOOD - output all openflow ports except the input port and those with flooding disabled via the OFPPC_NO_FLOOD port config bit (generally, this is done for STP)
 * OFPP_ALL -  output all openflow ports except the in port.
 * OFPP_CONTROLLER - Send to the controller.
 * OFPP_LOCAL - Output to local openflow port.
 * OFPP_NONE - Output to no where.

Enqueue
*******

Forwards a packet through the designated queue to implement rudimentary QoS behavior. See section of 5.2.2 of the OpenFlow spec.


.. code-block:: python

  class ofp_action_enqueue (object):
    def __init__ (self, **kw):
      self.port = 0
      self.queue_id = 0


* port (int) - must be a physical port
* queue_id (int) - specific queue id

Note that definition of queues is not a part of OpenFlow and is switch-specific.


Set VLAN ID
***********

If the packet doesn't have a VLAN header, this adds one and sets its ID to the specified value and its priority to 0.  If the packet already has a VLAN header, this just changes its ID.


.. code-block:: python

  class ofp_action_vlan_vid (object):
    def __init__ (self, **kw):
      self.vlan_vid = 0


* vlan_vid (int) - the ID to set the vlan id to (< 4094, of course)

Set VLAN priority
*****************

If the packet doesn't have a VLAN header, this adds one and sets its priority to the specified value and its ID to 0.  If the packet already has a VLAN header, this just changes its priority.


.. code-block:: python

  class ofp_action_vlan_pcp (object):
    def __init__ (self, **kw):
      self.vlan_pcp = 0



* vlan_pcp (short) - the priority to set the packet to (< 8)

Set Ethernet source or destination address
******************************************

Used to set the source or destination MAC (Ethernet) address.


.. code-block:: python

  class ofp_action_dl_addr (object):
    @classmethod
    def set_dst (cls, dl_addr = None):
      return cls(OFPAT_SET_DL_DST, dl_addr)
    @classmethod
    def set_src (cls, dl_addr = None):
      return cls(OFPAT_SET_DL_SRC, dl_addr)

    def __init__ (self, type = None, dl_addr = None):
      self.type = type
      self.dl_addr = EMPTY_ETH



* type (int) - either OFPAT_SET_DL_SRC or OFPAT_SET_DL_DST
* dl_addr (EthAddr) - the mac address to set.

It may be convenient to use the two class factory methods rather than directly creating an instance of this class.  For example, to create an action to rewrite the destination MAC address, you can use:


.. code-block:: python

  action = ofp_action_dl_addr.set_dst(EthAddr("01:02:03:04:05:06"))



Set IP source or destination address
************************************

Used to set the source or destination IP address.


.. code-block:: python

  class ofp_action_nw_addr (object):
    @classmethod
    def set_dst (cls, nw_addr = None):
      return cls(OFPAT_SET_NW_DST, nw_addr)
    @classmethod
    def set_src (cls, nw_addr = None):
      return cls(OFPAT_SET_NW_SRC, nw_addr)

    def __init__ (self, type = None, nw_addr = None):
      self.type = type
      if nw_addr is not None:
        self.nw_addr = IPAddr(nw_addr)
      else:
        self.nw_addr = IPAddr(0)



* type (int) - either OFPAT_SET_NW_SRC or OFPAT_SET_NW_DST
* nw_addr (IPAddr) - the IP address to set

As with MAC addresses, rather than constructing an instance of this class directly, it can be convenient to use the :mono:`set_src()` and :mono:`set_dst()` factory methods:


.. code-block:: python

  action = ofp_action_nw_addr.set_dst(IPAddr("192.168.1.14"))



Set IP Type of Service
**********************

Set the TOS field of an IP packet.


.. code-block:: python

  class ofp_action_nw_tos (object):
    def __init__ (self, nw_tos = 0):
      self.nw_tos = nw_tos


* nw_tos (short) - the tos of service to set.

Set TCP/UDP source or destination port
**************************************

Set the source or desintation TCP or UDP port.


.. code-block:: python

  class ofp_action_tp_port (object):
    @classmethod
    def set_dst (cls, tp_port = None):
      return cls(OFPAT_SET_TP_DST, tp_port)
    @classmethod
    def set_src (cls, tp_port = None):
      return cls(OFPAT_SET_TP_SRC, tp_port)

    def __init__ (self, type=None, tp_port = 0):
      self.type = type
      self.tp_port = tp_port



* type (int) - must be either OFPAT_SET_TP_SRC or OFPAT_SET_TP_DST
* tp_port (short) - the port value to set (< 65534)

As with the MAC and IP addresses, it may be convenient to use the two factory methods (:mono:`set_dst()` and :mono:`set_src()`) rather than explicitly creating instances of this class.


Example: Sending a FlowMod
**************************

To send a flow mod you must define a match structure (discussed above) and set some flow mod specific parameters as shown here:


.. code-block:: python

  msg = ofp_flow_mod()
  msg.match = match
  msg.idle_timeout = idle_timeout
  msg.hard_timeout = hard_timeout
  msg.actions.append(of.ofp_action_output(port = port))
  msg.buffer_id = <some buffer id, if any>
  connection.send(msg)


Using the connection variable obtained when the datapath joined, we can send the flowmod to the switch.


Example: Sending a PacketOut
****************************

In a similar manner to a flow mod, one must first define a packet out as shown here:


.. code-block:: python

  msg = of.ofp_packet_out(in_port=of.OFPP_NONE)
  msg.actions.append(of.ofp_action_output(port = outport))
  msg.buffer_id = <some buffer id, if any>
  connection.send(msg)


The inport is set to OFPP_NONE because the packet was generated at the controller and did not originate as a packet in at the datapath.


Nicira / Open vSwitch Extensions
================================

Open vSwitch supports a number of extensions to OpenFlow 1.0, and POX has growing support for these through the openflow.nicira module.  For example, there's support for multiple tables, Nicira Extensible Match, a number of the register-based actions, etc.

In general, if you want to use the Nicira extensions, you should put :mono:`openflow.nicira` on your commandline to run it like a component (or import it and call its launch() function directly).  You will then probably want to import it to get access to the classes and so forth that it contains.  In POX, the current convention is to import it as so: :mono:`import pox.openflow.nicira as nx` (however, this may change for the dart release).

Below, we discuss some aspects of POX's support for Nicira extensions, but please note that this is quite incomplete.  You might find it helpful to refer to the Open vSwitch documentation/source.  In particular, the `nicira-ext.h <https://raw.githubusercontent.com/openvswitch/ovs/master/include/openflow/nicira-ext.h>`_ header is useful, as is `this list of fields used in OVS <http://benpfaff.org/~blp/ovs-fields.pdf>`_ (which may not be official documentation, but I believe to have been written by one of OVS' primary authors).

.. todo:: Reference some of the other helpful OVS files.


Extended PacketIn Messages
**************************

OVS has an extended version of the packet-in message which contains the reason for the packet-in (e.g., whether it was because of a send-to-controller action or a table miss) and in the former case, the match of the relevant table entry.  This extended version is encapsulated inside an OpenFlow vendor message, and can be read via the generic vendor message hook mechanism or by handling the vendor event.  However, you can also have POX repurpose the normal PacketIn event and instead of having its :mono:`.ofp` attribute be a normal :mono:`ofp_packet_in`, it will be an :mono:`nxt_packet_in` instead.  To do this, pass the :mono:`--convert-packet-in` argument to the :mono:`openflow.nicira` component on the commandline.

Besides telling POX to treat these extended packet-ins as PacketIn events, you must also turn on the extended packet-in feature on the switches.  To do this, send a switch an :mono:`nx_packet_in_format` message.  Generally you'll do this in your ConnectionUp handler, like so:

.. code-block:: python

  event.connection.send(nx.nx_packet_in_format())


Multiple Table Support
**********************

While OpenFlow 1.0 only supports a single table, the Nicira extensions add support for multiple tables.  There are a couple aspects to this extension.

First, when manipulating the flow table, you must specify *which* table you mean.  The original ofp_flow_mod had no way to do this.  The extension repurposes eight bits of the sixteen bit "command" field to instead hold the table number.  POX's openflow.nicira includes a new ofp_flow_mod_table_id message type, which does this for you, adding a table_id attribute.  The new nx_flow_mod (see the Nicira Extended Match section for more on the latter) also includes this table_id attribute.  Note that before using this extended flow_mod, you must enable the extension, by sending an :mono:`nx_flow_mod_table_id` message, similar to with nx_packet_in_format mentioned above.

Additionally, while packets originally enter the first (zeroth) table, there is now an action which lets you send a packet to another table.  The easiest way to do this in POX is using a factory method of :mono:`nx_action_resubmit`:

.. code-block:: python

  flowmod.actions.append(nx.nx_action_resubmit.resubmit_table(table=42))


Flexible Flow Specifications (AKA Nicira Extended Match)
********************************************************

Nicira Extended Match (or NXM) is one of the more significant Nicira extensions, and is the basis for the OpenFlow Extensible Match (OXM) in OpenFlow 1.2.  Among other things, it allows for the matching of IPv6 fields, the flow cookie, metadata registers, and a whole slew of other things.

The core of NXM is the :mono:`nx_match` structure, which replaces the original :mono:`ofp_match` structure as the way to define matches for table entries.  Unlike :mono:`ofp_match`, which is just a fixed collection of fields, :mono:`nx_match` is really a flexible container for individual :mono:`nxm_entry`s.  In POX, its basic interface is similar to that of a normal Python list (though it should only contain match entries!).  Different types of :mono:`nxm_entry` are used to specify the attributes of packets you wish to match, such as addresses, IP protocol number, and so on.  There are :mono:`nxm_entry` types corresponding to each of the fixed fields in the original :mono:`ofp_match`, as well as a large number of new types.

Many nxm_entry types support *masks*.  For example, the IP source and destination address matching types (NXM_OF_IP_SRC and NXM_OF_IP_DST) support masks, which allows you to match subnets.  Unlike OVS, which only allows CIDR-compatible wildcarding of IP address bits, current versions of OVS allow for matching arbitrary netmasks via NXM.  Exactly which fields support masks and exactly which masks are supported is specific to the particular switch.  For example, earlier versions of OVS only allowed a few masks for Ethernet addresses, but current versions support arbitrary masks (this is a pattern – newer versions of OVS generally support more flexible masks for more fields).

The naming of the :mono:`nxm_entry` types correspond to their names in Open vSwitch.  Most of them start with ":mono:`NXM_`".  The types which correspond to the fixed fields in ofp_match start with ":mono:`NXM_OF_`".  Types which originate from Nicira start with ":mono:`NXM_NX_`". There are a few exceptions to the :mono:`NXM_` prefix.  As mentioned, NXM is the basis for OXM.  Most of the fields supported by OXM are also supported by NXM, and we use the NXM name.  However, there are some OXM entries which are supported by Open vSwitch which don't have an NXM equivalent.  For these, the :mono:`OXM_*` name is used and is available in :mono:`openflow.nicira`.  This may change in the future when POX actually supports OpenFlow 1.2+ (and therefore has direct support for OXM).

The best way I know to learn about the various :mono:`nxm_entry` types is by reading the sourcecode to nicira-ext.h (and possibly some other files) in Open vSwitch.  You can also look for them in POX's openflow/nicira.py code.  See the Additional Information subsection below for links.  At one point, POX supported most or all of the ones in OVS, though as OVS evolves, it's possible that some are added which missing from POX.  In general, they're easy to add yourself, and requests made to the mailing list will probably result in them being added as well.

Since the original :mono:`ofp_flow_mod` is specifically tied to the original :mono:`ofp_match`, the NXM extension also includes a new :mono:`nx_flow_mod` command to actually manipulate table entries that use extended matches.

Many entry types have *prerequisites*.  For example, if you want to match IPv4 addresses, you must first specify that the ethertype of the packet is, in fact, IPv4 (i.e., 0x0800).  **Order is significant here.**  As stated above, :mono:`nx_match`\'s basic interface similar to a Python list.  When an entry type has a prerequisites, the prerequisite entry must come first.  POX currently has no support for supplying these automatically or for checking the order or existence of these: if you screw it up, it's all on you to figure it out (though it may in the future).  Read the documentation on the entry type carefully!

.. note:: Perequisites have a relationship with OVS's "normal form".  The man page of OVS's ovs-ofctl has this to say: |brk| Flow descriptions should be in *normal form*. This means that a flow may only specify a value for an L3 field if it also specifies a particular L2 protocol, and that a flow may only specify an L4 field if it also specifies particular L2 and L3 protocol types. For example, if the L2 protocol type *dl_type* is wildcarded, then L3 fields *nw_src*, *nw_dst*, and *nw_proto* must also be wildcarded. Similarly, if *dl_type* or *nw_proto* (the L3 protocol type) is wildcarded, so must be *tp_dst* and *tp_src*, which are L4 fields.


Using nx_match
##############

There are multiple ways to use :mono:`nx_match` in POX.  The most straightforward interface is that it looks a bit like a Python list, containing (for example), :mono:`append()` and :mono:`insert()` methods which can be used to add individual entries.





.. code-block:: python
  :linenos:

  m = nx.nx_match()
  m.append( nx.NXM_OF_ETH_SRC(EthAddr("b8:fe:aa:6e:88:8c")) )


To include a mask, specify it as a second argument to the entry constructor:


.. code-block:: python
  :linenos:
  :lineno-start: 3

  # Only match broadcast/multicast packets
  m += nx.NXM_OF_ETH_DST("01:00:00:00:00:00", "01:00:00:00:00:00")


Note also in the above example, that we can skip the explicit usage of :mono:`EthAddr()`, and that we can use the += operator as an alternative to :mono:`append()`.

In addition to the list-like interface, all the built-in entry types magically have corresponding attributes on the nx_match object.  The property name is the name of the entry type, but lower case, and the leading :mono:`NXM`, :mono:`NXM_NX`, etc. prefixes are optional.  And, in fact, there are several of these pseudo-attributes.  An "un-suffixed" one, and ones with the suffixes ":mono:`_mask`", ":mono:`_with_mask`", and ":mono:`_entry`" for the value, the mask, the value+mask (as a tuple), and the actual :mono:`nxm_entry` object itself.  For example, line 2 above could also be represented as:


.. code-block:: python

  m.eth_src = "b8:fe:aa:6e:88:8c"



And line 4 could be one of the following:


.. code-block:: python

  m.eth_dst = "01:00:00:00:00:00"
  m.eth_dst_mask = "01:00:00:00:00:00"

  # .. or ...

  m.eth_dst_with_mask = ("01:00:00:00:00:00", "01:00:00:00:00:00")


Some even have special syntax.  For example, IP addresses with CIDR ranges (or CIDR-compatible netmasks) can use the shorthand:


.. code-block:: python

  m.of_ip_dst = "192.168.1.0/255.255.255.0"


(Note that you can use arbitrary, non-CIDR-compatible netmasks if you use one of the other forms!)




The :mono:`nx_flow_mod` is, in fact, a subclass of :mono:`ofp_flow_mod` – everything works pretty much the same, except that the match attribute is an :mono:`nx_match` instead of an :mono:`ofp_match`.


The Learn Action
****************

The learn action (:mono:`nx_action_learn`) is another of the powerful Nicira extensions which POX supports.  It allows for table entries to add new table entries.  A common reason for this is to do MAC learning on the switch without controller involvement (POX comes with an example of exactly this in the form of the :mono:`forwarding.l2_nx_self_learning` component). While the OVS docs/comments are the right place to learn about the learn action in general, we discuss it some here.

As stated above, the learn action causes the generation of a new table entry.  There is some *current* packet (which actually triggered the learn action).  This generates a new table entry for *future* packets.  The new table entry is defined by a series of flow_mod_specs which together create a "template" for the new table entry: its match and its actions.  Thus, there are two categories of flow_mod_specs: those which specify parts of the new entry's match, and those which specify actions for the new entry.

The match-oriented flow_mod_specs come in two flavors.  The simplest lets you specify a match criteria based on a hard-coded value (i.e., "future packets must have VLAN 100 to match this entry").  The second form uses a value in *current* packet to specify a match constraint for *future* packets (i.e., "future packets must have the same VLAN ID as the current packet to match this entry").  

The action-oriented flow_mod_specs come in four flavors.  Two of these generate OFPAT_OUTPUT actions on the new entry (which may be a real port number or may be some of the special "virtual" port numbers, e.g., OFPP_FLOOD).  The other two types both generate NXAST_REG_LOAD actions (i.e., header rewrites).  The difference between the two types of output and rewrite actions is the same as with the two variations of match entries: one uses hard-coded values ("set the VLAN ID to 101" or "output via port 3"), and the other uses a value from the current packet (e.g., "set the VLAN ID to be the same as the one in the current packet" or "output via the port stored in packet metadata register 3").

All of these variations can be seen as a combination of *source* and *destination*.  The source is either an immediate value, or its value in the current packet.  The destination is either a new match criterion, a field to rewrite, or an output.

POX provides two major ways of specifying flow specs – an explicit and list-oriented interface, and a shorthand method.  The following three examples are equivalent ways of creating a simple learning switch:


.. code-block:: python
  :linenos:

  # Straightforward.  List with flow_mod_spec constructor:
  learn = nx.nx_action_learn(table_id=1,hard_timeout=10)
  learn.spec = [
    nx.flow_mod_spec(src=nx.nx_learn_src_field(nx.NXM_OF_VLAN_TCI),
                     n_bits=12),
    nx.flow_mod_spec(src=nx.nx_learn_src_field(nx.NXM_OF_ETH_SRC),
                     dst=nx.nx_learn_dst_match(nx.NXM_OF_ETH_DST)),
    nx.flow_mod_spec(src=nx.nx_learn_src_field(nx.NXM_OF_IN_PORT),
                     dst=nx.nx_learn_dst_output())
  ]

  # Appending to list with flow_mod_spec factory:
  learn = nx.nx_action_learn(table_id=1,hard_timeout=10)
  fms = nx.flow_mod_spec.new # Just abbreviating this
  learn.spec.append(fms( field=nx.NXM_OF_VLAN_TCI, n_bits=12 ))
  learn.spec.append(fms( field=nx.NXM_OF_ETH_SRC, match=nx.NXM_OF_ETH_DST ))
  learn.spec.append(fms( field=nx.NXM_OF_IN_PORT, output=True ))

  # Shorthand flow_mod_spec chaining API:
  learn = nx.nx_action_learn(table_id=1,hard_timeout=10)
  learn.spec.chain(
      field=nx.NXM_OF_VLAN_TCI, n_bits=12).chain(
      field=nx.NXM_OF_ETH_SRC, match=nx.NXM_OF_ETH_DST).chain(
      field=nx.NXM_OF_IN_PORT, output=True)


There are some things to note in the above examples:

First, notice that fields are specified using their "NXM" entries (as described in the Nicira Extended Match section above).

Second, notice that we need not use the entire field – you can see that we specify n_bits to limit VLAN matching to 12 bits (since VLAN IDs are in fact only 12 bits of the VLAN TCI).  We can also specify a bit offset (as "ofs"); this wasn't necessary in the above case since it defaults to zero, and the VLAN ID starts at bit zero (had we wanted to match the VLAN priority, we'd have specified ofs=13 and n_bits=3).

Third, notice that while the first example is very explicit about the source and destination (in the sense mentioned just above), this is implicit in the other two.  The other two use keyword parameters based on the names of the various flow spec types: "field" and "immediate" specify the source, and "match", "load", or "output" specify the destination.

Fourth and lastly, notice that we can skip a "match" when it's the same as the "field".  This is just a little programmer-optimization for a pretty common case since we often want to match on values in the current packet, as with the VLAN ID.


Additional Information
**********************

We should add more documentation about using Nicira extensions.  For the moment, you might find the following to be useful references:

* The `forwarding.l2_nx <http://noxrepo.org/git/pox/blob/dart/pox/forwarding/l2_nx.py>`_ and `forwarding.l2_nx_self_learning <http://noxrepo.org/git/pox/blob/dart/pox/forwarding/l2_nx.py>`_ POX components.
* Open vSwitch's `nicira-ext.h <http://git.openvswitch.org/cgi-bin/gitweb.cgi?p=openvswitch;a=blob;f=include/openflow/nicira-ext.h>`_.  It has a lot of comments, and a lot of it applies to POX's implementation as well.
* The source code to POX's `openflow/nicira.py <http://noxrepo.org/git/pox/blob/dart/pox/openflow/nicira.py>`_.  It has some helpful comments.

About the OpenFlow Component's Initialization
=============================================

POX was significantly inspired by NOX, and NOX was unquestionably an OpenFlow controller.  Additionally, early versions of POX did not have any sort of dependency system.  These two factors led to POX having a special case: the OpenFlow component was enabled by default.  Later, the --no-openflow switch was added to disable OpenFlow when it wasn't needed or desired.  As time has gone on, usage of this switch has gone from useful on rare occasion to not entirely uncommon and it started to seem like an ugly annoyance.  However, simply not starting the OpenFlow component by default didn't seem like a good idea either: it would break many "known" commandlines, would be more typing for the still-very-common case where it's desired, and would likely cause many ugly startup exceptions until a fair amount of code was tweaked (since many components – even ones which use the dependency mechanism for other components – assume that the openflow component is always available).

There are a number of possible solutions to this problem and no permanent solution has yet been set in stone.  The currently favored high-level approach is to auto-load OpenFlow on demand.  A generic demand-loading mechanism was mostly written, but has not been merged.  Instead, the current solution in the dart branch (committed on October 13, 2013) has a number of changes specific to the OpenFlow component which attempt to detect whether the OpenFlow component is being used, and load it on demand if so.  The detection is not 100%: depending on how they're written, some older components will not trigger it, and thus will need the openflow component explicitly placed on the commandline.  Such components should be tweaked; ideally, they should use the dependency mechanism.  (l3_learning, for example, was modified to trigger autoloading.)

