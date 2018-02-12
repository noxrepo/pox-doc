POX APIs
--------

POX contains a number of APIs to help you develop network control applications.  Here, we attempt to describe some of them. It is certainly not exhaustive so feel free to contribute.


.. _`The POX Core object`:

Working with POX: The POX Core object
=====================================

POX has an object called "core", which serves as a central point for much of POX's API.  Some of the functions it provides are just convenient wrappers around other functionality, and some are unique.  However, one of the other major purposes of the core object is to provide a rendezvous between components.  Often, rather than using import statements to have one component import another component so that they can interact, components will instead "register" themselves on the core object, and other components will query the core object.  A major advantage to this approach is that the dependencies between components are not hard-coded, and different components which expose the same interface can be easily interchanged.  Thought of another way, this provides an alternative to Python's normal module namespace, which is somewhat easier to rearrange.

Many modules in POX will want to access the core object.  By convention, this is doing by importing the core object as so:


.. code-block:: python

 from pox.core import core


.. todo:: Write more about this!


Registering Components
**********************

As mentioned above, it can be convenient for a component to "register" an API-providing object on the core object.  An example of this can be found in the OpenFlow implementation – the POX OpenFlow component by default registers an instance of OpenFlowNexus as :mono:`core.openflow`, and other applications can then access a lot of the OpenFlow functionality from there.  There are basically two ways to register a component – using :mono:`core.register()` and using :mono:`core.registerNew()`.  The latter is really just a convenience wrapper around the former.

:mono:`core.register()` takes two arguments.  The second is the object we want to register on core.  The first is what name we want to use for it.  Here's a really simple component with a :mono:`launch()` function which registers the component as :mono:`core.thing`:


.. code-block:: python

  class MyComponent (object):
    def __init__ (self, an_arg):
      self.arg = an_arg
      print "MyComponent instance registered with arg:", self.arg

    def foo (self):
      print "MyComponent with arg:", self.arg


  def launch ():
    component = MyComponent("spam")
    core.register("thing", component)
    core.thing.foo() # prints "MyComponent with arg: spam"

In the case of, for example, :mono:`launch` functions which can be invoked multiple times, you may still only want to register an object once.  You could simply check if the component has already been registered (using :mono:`core.hasComponent()`), but this can also be done with :mono:`core.registerNew()`.  While you pass a specific object to :mono:`core.register()`, you pass a *class* to :mono:`core.registerNew()`.  If the named component has already been registered, :mono:`registerNew()` just does nothing.

:mono:`registerNew()` generally takes a single parameter – the class you want it to instantiate.  If that class's:mono:` __init__` method takes arguments, you can pass them as additional parameters to :mono:`registerNew()`.  For example, we might change the :mono:`launch` function above to:


.. code-block:: python

  def launch ():
    core.registerNew(MyComponent, "spam")
    core.MyComponent.foo() # prints "MyComponent with arg: spam"

Note that :mono:`registerNew()` automatically registers the given object using the object's class name (that is, it's now "MyComponent" instead of "thing").  This can be overridden by giving the object an attribute called :mono:`_core_name`:

.. code-block:: python

  class MyComponent (object):
    _core_name = "thing"

    def __init__ (self, an_arg):
      self.arg = an_arg
      print "MyComponent instance registered with arg:", self.arg

    def foo (self):
      print "MyComponent with arg:", self.arg



Dependency and Event Management
*******************************

When components in POX are dependent on other components (i.e., objects registered on core), it's often (though not always) because they want to listen to events of that other component.  POX's contains a useful function which makes it pretty easy to both "depend" on another component in a sane way and also to set up event handlers for you: :mono:`core.listen_to_dependencies()`.

:mono:`listen_to_dependencies`'s arguments:

.. code-block:: python

  sink, components=None, attrs=True, short_attrs=False, listen_args={}

And here's its docstring more or less verbatim:

  Look through :mono:`sink` for handlers named like :mono:`_handle_<ComponentName>_<EventName>`. Use that to build a list of components, and append any components explicitly specified by :mono:`components`.

  :mono:`listen_args` is a dict of :mono:`"component_name":{"arg_name":"arg_value",...}`, allowing you to specify additional arguments to :mono:`addListeners()`.

  When all the referenced components are registered, do the following:

  #. Set up all the event listeners
  #. Call :mono:`._all_dependencies_met()` on :mono:`sink` if it exists
  #. If :mono:`attrs=True`, set attributes on :mono:`sink` for each component (e.g, :mono:`sink._openflow_` would be set to core.openflow)

  For example, if "topology" is a dependency, a handler for topology's :mono:`SwitchJoin` event must be defined as so:

  .. code-block:: python

    def _handle_topology_SwitchJoin (self, ...):


If dependencies specified in this fashion are not resolved during POX's startup phase, a message is logged about POX still waiting on some component (e.g., "core:Still waiting on 1 component(s)").  The debug-level log will contain more detailed information on this subject.  See the FAQ entry on this subject.


Working with Addresses: pox.lib.addresses
=========================================

IPv4, IPv6, and Ethernet addresses in POX are represented by the IPAddr, IPAddr6, and EthAddr classes of pox.lib.addresses.  In some cases, other address formats may work (e.g., dotted-quad IP addresses), but using the address classes should *always* work.

For example, when working with IP addresses:

.. code-block:: python

  from pox.lib.addresses import IPAddr, IPAddr6, EthAddr

  ip = IPAddr("192.168.1.1")
  print str(ip) # Prints "192.168.1.1"
  print ip.toUnsignedN() # Convert to network-order unsigned integer -- 16885952
  print ip.raw # Returns a length-four bytes object (a four byte string, more or less)

  ip = IPAddr(16885952,networkOrder=True)
  print str(ip) # Also prints "192.168.1.1" !


pox.lib.addresses also contains various utility functions for parsing netmasks, CIDR notation, checking whether an IP is within a specific subnet, and so on.


The Event System: pox.lib.revent
================================

Event Handling in POX fits into the publish/subscribe paradigm.  Certain objects publish events (in revent lingo, this is "*raising*" an event; also sometimes called "sourcing", "firing" or "dispatching" an event).  One can then subscribe to specific events on these objects (in revent lingo, this is "*listening* *to*"; sometimes also "handling" or "sinking"); what we mean by this is that when the event occurs, we'd like a particular piece of code to be called (an "*event handler*" or sometimes an "event listener").  (If there's one thing we can say about events, it's that there's no shortage of terminology.)

.. note:: The revent library can actually do some weird stuff. POX only uses a fairly non-weird subset of its functionality, and mostly uses a pretty small subset of that subset!  What is described in this section is the subset that POX makes use of most heavily.

Events in POX are all instances of subclasses of :mono:`revent.Event`.  A class that raises events (an event source) inherits from :mono:`revent.EventMixin`, and declares which events it raises in a class-level variable called :mono:`_eventMixin_Events`.  Here's an example of a class that raises two events:

.. code-block:: python

  class Chef (EventMixin):
    """
    Class modeling a world class chef
   
    This chef only knows how to prepare spam, but we assume does it really well.
    """
    _eventMixin_events = set([
      SpamStarted,
      SpamFinished,
    ])



Handling Events
***************

So perhaps your program has an object of class :mono:`Chef` called :mono:`chef`.  You know it raises a couple events.  Maybe you're interested in when your delicious spam is ready, so you'd like to listen to the :mono:`SpamFinished` event.


Event Handlers
##############

First off, let's see exactly what an event listener looks like.  For one thing: it's a function (or a method or some other thing that's callable).  They almost always just take a single argument – the event object itself (though this isn't *always* the case – an event class can change this behavior, in which case, its documentation should mention it!).  Assuming :mono:`SpamFinished` is a typical event, it might have a handler like:

.. code-block:: python

  def spam_ready (event):
    print "Spam is ready!  Smells delicious!"



Listening To an Event
#####################

Now we need to actually set our :mono:`spam_ready` function to be a listener for the :mono:`SpamFinished` event:

.. code-block:: python

  chef.addListener(SpamFinished, spam_ready)


Sometimes you may not have the event class (e.g., :mono:`SpamFinished`) in scope.  You can import it if you want, but you can also use the :mono:`addListenerByName()` method instead:

.. code-block:: python

  chef.addListenerByName("SpamFinished", spam_ready)



Automatically Setting Listeners
###############################

Often, your event listener is a method on a class.   Also, you often are interested in listening to multiple events from the same source object.  revent provides a shortcut for this situation: :mono:`addListeners()`.

.. code-block:: python

  class HungryPerson (object):
    """ Models a person that loves to eat spam """

    def __init__ (self):
      chef.addListeners(self)

    def _handle_SpamStarted (self, event):
      print "I can't wait to eat!"

    def _handle_SpamFinished (self, event):
      print "Spam is ready!  Smells delicious!"


When you call :mono:`foo.addListeners(bar)`, it looks through the events of :mono:`foo`, and if it sees a method on :mono:`bar` with a name like :mono:`_handle_*EventName*`, it sets that method as a listener.

In some cases, you may want to have a single class listening to events from multiple event sources.  Sometimes it's important that you can tell the two apart.  For this purpose, you can also use a "prefix" which gets inserted into the handler names:

.. code-block:: python

  class VeryHungryPerson (object):
    """ Models a person that is hungry enough to need two chefs """

    def __init__ (self):
      master_chef.addListeners(self, prefix="master")
      backup_chef.addListeners(self, prefix="secondary")

    def _handle_master_SpamFinished (self, event):
      print "Spam is ready!  Smells delicious!"

    def _handle_secondary_SpamFinished (self, event):
      print "Backup spam is ready.  Smells slightly less delicious."



Creating Your Own Event Types
*****************************

As noted above, events are subclasses of :mono:`revent.Event`.  So to create an event, simply create a subclass of :mono:`Event`. You can add any extra attributes or methods you want.  Continuing our example:

.. code-block:: python

  class SpamStarted (Event):
    def __init__ (self, brand = "Hormel"):
      Event.__init__(self)
      self.brand = brand

    @property
    def genuine (self):
      # If it's not Hormel, it's just canned spiced ham!
      return self.brand == "Hormel"


Note that you should explicitly call the superclass's :mono:`__init__()` method!  (You can do this as above, or using the new-school :mono:`super(MyEvent, self).__init__()`.)

.. note:: In newer versions of POX, calling the superclass __init__() is no longer required.

Voila! You can now raise new instances of your event!

Note that in our handlers for :mono:`SpamStarted` events, we could have accessed the :mono:`brand` or :mono:`genuine` attributes on the event object that gets passed to the handler.

*Note:* While revent doesn't care what you name your event classes, if you are using POX's :mono:`listen_to_dependencies()` mechanism (described below), the class names must not contain an underscore (which is consistent with POX naming style, described in a later section).


Raising Events
**************

To raise an event so that listeners are notified, you just call :mono:`raiseEvent` on the object that will publish the event:

.. code-block:: python

  # One way to do it
  chef.raiseEvent(SpamStarted("Generic"))

  # Another way (slightly preferable)
  chef.raiseEvent(SpamStarted, "Generic")


(The second way is slightly preferable because if there are no listeners, it avoids ever even creating the event object.)

Often, a class will raise events on itself (:mono:`self.raiseEvent(...)`), but as you see in the example above, this isn't *necessarily* the case.

There is a variant of :mono:`raiseEvent()` called :mono:`raiseEventNoErrors()`.  This behaves much the same as :mono:`raiseEvent()`, but exceptions in event handlers are caught automatically.


Binding to Components' Events
*****************************

Often, event sources are "components" in that they've registered an object on the POX core object, and it's that object which sources events you want to listen to.  While you can certainly use the above methods for adding listeners, the core object also has the useful :mono:`listen_to_dependencies()` method, which is documented in the section `The POX Core object`_.


Advanced Topics in Event Handling
*********************************

Events With Multiple Listeners
##############################

Above, we described the basics for handling events and demonstrated how to set listeners.  A given source and event can have any number of listeners – you don't have to do anything special to support that.  Here, however, we should discuss two issues which sometimes pop up when there are multiple listeners (often together).

The first of these is: when there are multiple listeners, in what order are they called?  By default, the answer is that it's undefined.  However, this can be overridden by specifying :mono:`priority` in your call to :mono:`addListener()` or :mono:`addListeners()`.  Priorities should be integers; higher numbers mean call this listener sooner.  Listeners with no priority set are equivalent to priority 0 – you can use :mono:`negative` priorities to be called after these.

This brings us to the second issue: halting events.  When an event handler is invoked, it has an opportunity to halt the event – stopping further handlers from being invoked for that method.  This is generally used sort of like a filter: a higher priority handler sees the event first, halts the event if it handles it, or (if it doesn't handle it) allows a later listener to handle it.  This is the mechanism that the :mono:`mac_blocker` component uses, for example: it halts :mono:`PacketIn` events for blocked addresses, but allows them to pass to a forwarding component for unblocked addresses.  To halt an event, you may either set the :mono:`.halt` attribute of the event object to :mono:`True`, or have the listener return :mono:`EventHalt` (or :mono:`EventHaltAndRemove`; see below).  In the latter case, you'll need to import :mono:`EventHalt` from :mono:`pox.lib.revent`.


Removing Listeners and One-Time Events
######################################

In many cases, it's sufficient to set up a listener and forget about it forever.  However, it is sometimes the case that you want to stop listening to an event.  There are a few different ways to do this.

The first way is very similar to halting an event as described above: the handler just returns :mono:`EventRemove` (or :mono:`EventHaltAndRemove`).  Of course, this can only be done from inside the handler at the time an event as actually being handled.  If you'd like to remove the handler from outside the handler, you can use the :mono:`removeListener()` method.  The easiest way to use this is simply to pass it an "event ID".  Although not discussed earlier, this is the return value of :mono:`addListener()`.  Thus, you can easily save the return value from :mono:`addListener()` and pass it to :mono:`removeListener()` later to unhook the handler.  Things work pretty much as you'd hope for :mono:`addListeners()` as well – it returns a sequence of event IDs, which can be simply passed to :mono:`removeListeners()`.

revent contains a special shortcut for a fairly common case: when you care about an event only the first time it fires.  Simply pass :mono:`once=True` into :mono:`addListener()`, and the listener is automatically removed after the first time it's fired.


Weak Event Handlers
###################

By default, when a listening to an event source, this creates a reference to the source.  Generally, this means that the lifetime of the event source is now bound to the lifetime of the event handler.  Often this is just fine (and even desirable).  However, there are exceptions.  To provide for these exceptions, one can pass :mono:`weak=True` into :mono:`addListener()`.  This creates a weak reference: if the source object has no other references, the listener is removed automatically.


Working with packets: pox.lib.packet
====================================

Lots of applications in POX interact with packets (e.g., you might want to construct packets and send them out of a switch, or you may receive them from a switch via an :mono:`ofp_packet_in` OpenFlow message).  To facilitate this, POX has a library for parsing and constructing packets.  The library has support for a number of different packet types.

Most packets have some sort of a header and some sort of a payload.  A payload is often another type of packet.  For example, in POX one generally works with :mono:`ethernet` packets which often contain :mono:`ipv4` packets (which often contain :mono:`tcp` packets...).  Some of the packet types supported by POX are:

* ethernet
* ARP
* IPv4
* ICMP
* TCP
* UDP
* DHCP
* DNS
* LLDP
* VLAN

All packet classes in POX are found in pox/lib/packet.  By convention, you import the POX packet library as:


.. code-block:: python

  import pox.lib.packet as pkt


One can navigate the encapsulated packets in two ways: by using the :mono:`payload` attribute of the packet object, or by using its :mono:`find()` method. For example, here is how you could parse an ICMP message using the :mono:`payload` attribute:

.. code-block:: python

  def parse_icmp (eth_packet):
    if eth_packet.type == pkt.IP_TYPE:
      ip_packet = eth_packet.payload
      if ip_packet.protocol == pkt.ICMP_PROTOCOL:
        icmp_packet = ip_packet.payload
  ...


This is probably not the best way to navigate a packet, but it illustrates the structure of packet headers in POX. At each level of encapsulation the packet header values can be obtained. For example, the source ip address of the ip packet above and ICMP sequence number can be obtained as shown:

.. code-block:: python

  ...
  src_ip = ip_packet.srcip
  icmp_code = icmp_packet.code


And similarly for other packet headers. Refer to the specific packet code for other headers.

A packet object's :mono:`find()` method can be used to find a specific encapsulated packet by the desired type name (e.g., :mono:`"icmp"`) or its class (e.g., :mono:`pkt.ICMP`).  If the packet object does not encapsulate a packet of the requested type, :mono:`find()` returns None.  For example:

.. code-block:: python

  def handle_IP_packet (packet):
    ip = packet.find('ipv4')
    if ip is None:
      # This packet isn't IP!
      return
    print "Source IP:", ip.srcip


The following sections detail *some* of the useful attributes/methods/constants for *some* of the supported packet types.


Ethernet (ethernet)
*******************

Attributes:

* dst (EthAddr)
* src (EthAddr)
* type (int) - The ethertype or ethernet length field.  This will be 0x8100 for frames with VLAN tags
* effective_ethertype (int) - The ethertype or ethernet length field.  For frames with VLAN tags, this will be the type referenced in the VLAN header.

Constants:

* IP_TYPE, ARP_TYPE, RARP_TYPE, VLAN_TYPE, LLDP_TYPE, JUMBO_TYPE, QINQ_TYPE - A variety of ethertypes

Pretty-print ethertype as string:

.. code-block:: python

  pkt.ETHERNET.ethernet.getNameForType(packet.type)







IP version 4 (ipv4)
*******************

Attributes:

* srcip (IPAddr)
* dstip (IPAddr)
* tos (int) - 8 bits of Type Of Service / DSCP+ECN
* id (int) - identification field
* flags (int)
* frag (int) - fragment offset
* ttl (int)
* protocol (int) - IP protocol number of payload
* csum (int) - checksum

Constants:

* ICMP_PROTOCOL, TCP_PROTOCOL, UDP_PROTOCOL - Various IP protocol numbers
* DF_FLAG - Don't Fragment flag bit
* MF_FLAG - More Fragments flag bit

TCP (tcp)
*********

Attributes:

* srcport (int) - Source TCP port number
* dstport (int) - Destination TCP port number
* seq (int) - Sequence number
* ack (int) - ACK number
* off (int) - offset
* flags (int) - Flags as bitfield (easier to use all-uppercase flag attributes)
* csum (int) - Checksum
* options (list of tcp_opt objects)
* win (int) - window size
* urg (int) - urgent pointer
* FIN (bool) - True when FIN flag set
* SYN (bool) - True when SYN flag set
* RST (bool) - True when RST flag set
* PSH (bool) - True when PSH flag set
* ACK (bool) - True when ACK flag set
* URG (bool) - True when URG flag set
* ECN (bool) - True when ECN flag set
* CWR (bool) - True when CWR flag set

Constants:

* FIN_flag, SYN_flag, etc. - Bits corresponding to flags

tcp_opt class
#############

Attributes:

* type (int) - TCP Option ID (probably corresponding constant below)
* val (varies) - Option value

Constants:

* EOL, NOP, MSS, WSOPT, SACKPERM, SACK, TSOPT - Option type IDs

Example: ARP messages
*********************

You might want the controller to proxy the ARP replies rather than flood them all over the network depending on whether you know the MAC address of the machine the ARP request is looking for. To handle ARP packets in you should have an event listener set up to receive packet ins as shown:

.. code-block:: python

  def _handle_PacketIn (self, event):
    packet = event.parsed
    if packet.type == packet.ARP_TYPE:
      if packet.payload.opcode == arp.REQUEST:
        arp_reply = arp()
        arp_reply.hwsrc = <requested mac address>
        arp_reply.hwdst = packet.src
        arp_reply.opcode = arp.REPLY
        arp_reply.protosrc = <IP of requested mac-associated machine>
        arp_reply.protodst = packet.payload.protosrc
        ether = ethernet()
        ether.type = ethernet.ARP_TYPE
        ether.dst = packet.src
        ether.src = <requested mac address>
        ether.payload = arp_reply
        #send this packet to the switch
        #see section below on this topic
      elif packet.payload.opcode == arp.REPLY:
        print "It's a reply; do something cool"
      else:
        print "Some other ARP opcode, probably do something smart here"


See the :mono:`l3_learning` component for a more complete example of using the controller to parse ARP requests and generate replies.


Constructing Packets from Scratch and Reading Packets from the Wire
*******************************************************************

The above examples have mostly focused on working with the packet objects.  While those are convenient for working with in Python, they're not the form packets actually take when they're being sent or received over a network – at that level, the packets are all really just a sequence of bytes.

To go from a packet object (which possibly contains other packet objects as its payload) to its on-the-wire format (a series of bytes), you call the object's .pack() method.  To do the inverse, you call the appropriate packet type's .unpack() *class method*. If you're working with PacketIn objects, this is done for you automatically – the event's .parsed property will contain the packet objects.  However, there are cases where you'll want to do it yourself.  For example, if you are reading from the file descriptor side of a TUN interface, you may want to parse out the IP packets you read.  In this case, you'd use pkt.ipv4.unpack(<your data>).

You also may need to manually unpack things when you've got cases the packet library doesn't understand.  For example, the packet library understands that an IPv4 packet might contain ICMP or TCP (and a few others).  As of this writing, it does *not* understand either flavor of IP-in-IP encapsulation (protocol 4 or protocol 94).  When the packet library doesn't understand how to parse a packet's payload, it simply includes it as raw bytes.  Thus, IP-in-IP comes out of the packet library as something like ethernet->ipv4->raw_data.  If you want work with the encapsulated IPv4 packet, you'll have to unpack it yourself: inner_ip = pkt.ipv4.unpack(outer_eth.find('ipv4').payload).


Threads, Tasks, and Timers: pox.lib.recoco
==========================================

This is a big subject, and a lot could be said.  Feel free to add something!

POX's recoco library is for implementing simple cooperative tasks.  Perhaps the major benefit of cooperative tasks is that you generally don't need to worry much about synchronization between them.

There's a small amount of example material in :mono:`pox/lib/recoco/examples.py`

The first rule of recoco tasks: don't block.  Stalling a recoco task (that is, stalling the scheduler's thread) will keep other tasks from running.  Some blocking operations (sleep, select, etc.) have recoco-friendly equivalents – see recoco's source or reference for details.


Using Normal Threads
********************

You can use recoco, but you don't have to -- you can use normal threading if you want.  Indeed, there are several parts of POX which use normal threads (the web server, for example).  While you don't need to worry much about synchronization between recoco tasks, you do need to think about synchronization between recoco task and normal threads.  Often, it's reasonable to start up a worker thread, and when it's done, have it fire a method using core.callLater() or have it schedule a recoco Task (using the threadsafe non-fast scheduling function).


Executing Code in the Future using a Timer
******************************************

It's often useful to have a piece of code execute from time to time.  For example, you may want to examine the bytes transferred over a specific flow every 10 seconds.  It's also a fairly common case where you know you want something to happen at some specific time in the future; for example, if you send a barrier, you might want to disconnect the switch if 5 seconds go by without the barrier reply showing up.  This is the type of task that the pox.lib.recoco.Timer class is designed to handle – executing a piece of code at a single or recurring time in the future.

*Note:* The POX core object's *callDelayed()* is often an easier way to set a simple timer.  (See example below.)

Timer Constructor Arguments
###########################

=============== ========================= =================================================================================================================================================================================================================================================================
arg             type - default            meaning
=============== ========================= =================================================================================================================================================================================================================================================================
_timeToWake_    number (seconds)          Amount of time to wait before calling callback (absoluteTime = False), or specific time to call callback (absoluteTime = True)
_callback_      callable (e.g., function) A function to call when the timer elapses
_absoluteTime_  boolean - False           When False, timeToWake is a number of seconds in the future. When True, timeToWake is a specific time in the future (e.g., a number of seconds since the epoch, as reported with time.time()). Note that absoluteTime=True can not be used with recurring timers.
_recurring_     boolean - False           When False, the timer online fires once - timeToWake seconds from when it's started. When True, the timer fires every timeToWake seconds.
_args_, _kw_    sequence, dict - empty    These are arguments and keyword arguments passed to _callback_.
_scheduler_     Scheduler - None          The scheduler this timer is executed with. None means to use the default (you want this).
_started_       boolean - True            If True, the timer is started automatically.
_selfStoppable_ boolean - True            If True, the callback of a recurring timer can return False to cancel the timer.
=============== ========================= =================================================================================================================================================================================================================================================================

Timer Methods
#############

====== ========= ===============================================
name   arguments meaning
====== ========= ===============================================
cancel None      Stop the timer (do not call the callback again)
====== ========= ===============================================

Example - One-Shot timer
########################

.. code-block:: python

  from pox.lib.recoco import Timer

  def handle_timer_elapse (message):
    print "I was told to tell you:", message

  Timer(10, handle_timer_elapse, args = ["Hello"])

  # Prints out "I was told to tell you: Hello" in 10 seconds

  # Alternate way for simple timers:
  from pox.core import core # Many components already do this
  core.callDelayed(10, handler_timer_elapse, "Hello") # You can just tack on args and kwargs.


Example - Recurring timer
#########################

.. code-block:: python

  # Simulate a long road trip

  from pox.lib.recoco import Timer

  we_are_there = False

  def are_we_there_yet ():
    if we_are_there: return False # Cancels timer (see selfStoppable)
    print "Are we there yet?"

  Timer(30, are_we_there_yet, recurring = True)



Working with sockets: ioworker
==============================

pox.lib.ioworker contains a high level API for working with asynchronous sockets in POX.  Sends are fire-and-forget, received data is buffered and a callback fired when there's some available, etc.

.. todo:: Documentation and samples


Working with pcap/libpcap: pxpcap
=================================

pxpcap is POX's pcap library.  It was written because we couldn't find an existing pcap library for Python which provided all of the following:

#. was maintained
#. supported Windows, Linux, and MacOS
#. supported both capture and injection
#. could capture at a reasonable rate

Along with meeting these goals, pxpcap exposes other pcap and pcap-related functionality, such as enumerating network interfaces, and reading/writing tcpdump/pcap trace files.

The pxpcap directory also contains a couple small utility POX components which can serve as examples if you want to write your own code using pxpcap.  The most obvious of these could be called "pxshark" – it captures traffic from an interface, dissects it using the POX packet library, and dumps the results.  You can run this like so:


.. code-block:: none

  ./pox.py pox.lib.pxpcap --interface=eth0



Building pxpcap
***************

pxpcap is written partially in C++ and partially in Python.  If you wish to use all of its features, you must build the C++ portion (the pure Python parts should work regardless).  Its directory has scripts to make it on Windows, Mac OS, and Linux.  It requires that you have a C++ compiler and libpcap/winpcap development files installed.  Beyond that, building it should be fairly straightforward; something like the following:


.. code-block:: none

  cd pox/lib/pxpcap/pxpcap_c
  ./build_linux # or ./build_mac or build_win.bat


See the following subsections for tips on specific troublesome configurations.

Note that the setuptools script was originally intended to allow pxpcap to be used either with out without the rest of POX.  However, keeping it usable without POX has not been a high priority.  Feel free to pitch in here!

Also note that the C portion is required for the POX datapath (software switch) to forward traffic on real interfaces.


Using pxpcap with older versions of Python
******************************************

pxpcap has a mode where it uses Python's bytearray C API, which is relatively new (meaning not particularly new at all).  If you're running on recent Python 2.7 (the recommended configuration for POX), this will certainly not be a problem.  If you are trying to use pxpcap with some old Python, you can disable the bytearray mode by passing -DNO_BYTEARRAYS to the compiler.  This isn't currently very well supported and you'll probably need to tweak the setuptools script yourself.


Using pxpcap with PyPy
**********************

If you're using the normal CPython interpreter, you can safely ignore this section.  If you're using PyPy, the good news is that pxpcap can be made to work (at least for PyPy 1.9+).  The bad news is that the build scripts are questionable.  On Mac OS, the setuptools script seems to build it okay, though the simple install script doesn't work right since PyPy names its extensions differently, and you'll have to copy the .so to the pxpcap directory yourself (or you could try installing it globally).  On Linux, my (Murphy's) experience is that the setuptools script doesn't even work right.  I just built it by hand (adjust the output name in the following if you're not using PyPy 2.1):


.. code-block:: none

  g++ pxpcap.cpp -I /home/pox/pypy/include/ -DNO_BYTEARRAYS -DHAVE_PCAP_GET_SELECTABLE_FD -lpcap -shared -fPIC -o ../pxpcap.pypy-21.so


The other caveat is that pxpcap's bytearray mode (where captured data is put into a bytearray instead of a bytes object) is not supported in PyPy, and you get bytes instead of a bytearray no matter what you do.



