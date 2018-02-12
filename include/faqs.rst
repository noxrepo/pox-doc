FAQs
----

What versions of OpenFlow does POX support?
===========================================

POX currently supports OpenFlow 1.0.  It also supports a number of the Nicira / Open vSwitch ("nx") extensions (many of which are the basis for features in later OpenFlow versions).

Search the mailing list for a partial port to OpenFlow 1.1.

We'll probably get around to supporting later versions eventually, or would be happy to work with others on getting patches mainlined.  Come to the mailing list and raise the subject if you're interested.


What does the ``Fields ignored due to unspecified prerequisites`` warning mean?
===============================================================================

See the next question.


I tried to install a table entry but got a different one.  Why?
===============================================================

This question also presents itself as "What does the ``Fields ignored due to unspecified prerequisites`` warning mean?"

Basically this means that you specified some higher-layer field without specifying the corresponding lower-layer fields also.  For example, you may have tried to create a match in which you specified only :mono:`tp_dst=80`, intending to capture HTTP traffic.  You can't do this.  To match TCP port 80, you must also specify that you intend to match TCP (:mono:`nw_proto=6`).  And in order to match on TCP, you must also match on IP (:mono:`dl_type=0x800`).

For more information, see the text on "normal form" flow descriptions in the :mono:`ovs-ofctl` man page, or the new clarifying text added to section 3.4 in the OpenFlow 1.0.1 specification.


What is a "datapath"?  What is a DPID?
======================================


More or less, a datapath is a logical OpenFlow switch.  A physical "OpenFlow switch" – meaning a box with ethernet ports – may have more than one datapath (though usually won't).

A DPID is a datapath identifier, and is part of the OpenFlow specification, though the OpenFlow specification calls it a datapath_id, and is pretty vague about it in general.  Basically, it is a unique identifier for a switch so that someone/something (e.g., an OpenFlow controller) can uniquely identify the switch.  During the initial handshake, a switch sends its DPID to the controller as part of the :mono:`ofp_switch_features` message.

A DPID is 64 bits.  The spec claims the lower 48 bits are *intended* to be the switch's ethernet address.  This statement has always been a bit confusing -- traditionally, a switch isn't an endpoint, so what addresses does a switch even really have?  The only answer that makes sense to this author is that it's the ethernet address associated with the IP address used for the OpenFlow control channel.  But in implementations, this is not always true because the OpenFlow control channel often may originate from one of several interfaces and it'll use whatever ethernet address goes with it.

In practice, its pretty arbitrary, and often user-configurable independent of any ethernet address.  It's probably a decent idea to always just treat it as an opaque value which should be unique.  If a vendor happens to base it on some particular ethernet address, treat that as an implementation detail for how they achieve uniqueness and not as there being any sort of real relationship between the two.

To give a more concrete answer: with Open vSwitch, it defaults to the ethernet address of the switch's "local" port with the top 16 bits zeroed (this ethernet address being generated at random when last I checked).  With Mininet, this generally gets overridden to match the number of the Mininet switch.

See the first few sections of the "OpenFlow in POX" section of this manual for more.


How do I create a firewall / block TCP ports?
=============================================

An easy way to do this is to use a forwarding component which does fine-grained flows (e.g., :mono:`l2_learning`), and then intercept the :mono:`PacketIn` events.  When you see a packet you want to block, kill the event.  This will keep :mono:`l2_learning` from seeing the event and installing a flow for it.  The :mono:`mac_blocker` component pretty much works like this.  Here's a simple example for blocking arbitrary TCP ports:

.. code-block:: python

  """
  Block TCP ports

  Save as ext/blocker.py and run along with l2_learning.

  You can specify ports to block on the commandline:
  ./pox.py forwarding.l2_learning blocker --ports=80,8888,8000

  Alternatively, if you run with the "py" component, you can use the CLI:
  ./pox.py forwarding.l2_learning blocker py
   ...
  POX> block(80, 8888, 8000)
  """

  from pox.core import core

  # A set of ports to block
  block_ports = set()

  def block_handler (event):
    # Handles packet events and kills the ones with a blocked port number

    tcpp = event.parsed.find('tcp')
    if not tcpp: return # Not TCP
    if tcpp.srcport in block_ports or tcpp.dstport in block_ports:
      # Halt the event, stopping l2_learning from seeing it
      # (and installing a table entry for it)
      core.getLogger("blocker").debug("Blocked TCP %s <-> %s",
                                      tcpp.srcport, tcpp.dstport)
      event.halt = True

  def unblock (*ports):
    block_ports.difference_update(ports)

  def block (*ports):
    block_ports.update(ports)

  def launch (ports = ''):

    # Add ports from commandline to list of ports to block
    block_ports.update(int(x) for x in ports.replace(",", " ").split())

    # Add functions to Interactive so when you run POX with py, you
    # can easily add/remove ports to block.
    core.Interactive.variables['block'] = block
    core.Interactive.variables['unblock'] = unblock

    # Listen to packet events
    core.openflow.addListenerByName("PacketIn", block_handler)


How can I change the OpenFlow port from 6633?
=============================================

If you turn on verbose logging, you'll see that it's the openflow.of_01 module which listens for connections.  That's the hint: it's this component that you need to reconfigure.  Do so by passing a "port" argument to this component on the commandline:

.. code-block:: bash

 ./pox.py openflow.of_01 --port=1234 <other commandline arguments>

How can I have some components start automatically every time I run POX?
========================================================================

The short answer is that there's no supported way for doing this.  *However*, it's pretty simple to just create a small component that launches whatever other components you want.

For example, let's say you were tired of always having to remember the following commandline:

.. code-block:: bash

 ./pox.py log.level --DEBUG samples.pretty_log openflow.keepalive --interval=15 forwarding.l2_pairs

By writing a simple component, you can replace with above with simply:

.. code-block:: bash

 ./pox.py startup

The code for the simple component should be placed in :mono:`ext/startup.py` and would contain the following code:

.. code-block:: python

  # Put me in ext/startup.py

  def launch ():
    from pox.log.level import launch
    launch(DEBUG=True)

    from pox.samples.pretty_log import launch
    launch()

    from pox.openflow.keepalive import launch
    launch(interval=15) # 15 seconds

    from pox.forwarding.l2_pairs import launch
    launch()

.. note:: Note that here, as elsewhere in POX, it's important to import POX modules using their full name – including the leading "pox" package.  Not doing so can lead to confusing and incorrect behavior.



How do I get switches to send complete packet payloads to the controller?
=========================================================================

By default, when a packet misses all the entries in the flow table, only the first X bytes of the packet are sent to the controller.  In POX, this defaults to :mono:`OFP_DEFAULT_MISS_SEND_LEN`, which is 128.  This is probably enough for, e.g., ethernet, IP, and TCP headers... but probably not enough for complete packets.  If you want to inspect complete packet payloads, you have two options:

 1. *Install a table entry.*  If you install a table entry with a :mono:`packet_out` to :mono:`OFPP_CONTROLLER`, POX will have the switch send the complete packet by default (you can manually set some smaller number of bytes if you want).
 2. *Change the miss send length.* If you set :mono:`core.openflow.miss_send_len` during startup (before any switches connect), switches should send that many bytes when a packet misses the whole table. Check the :mono:`info.packet_dump` and :mono:`misc.full_payload` components for examples.


How can I communicate between components?
=========================================

Components are just Python packages and modules, so one way you can do this is the same way you communicate between any Python modules -- import one of them and access its top-level variables and functions.

POX also has an alternate mechanism, which is described more in the section `Working with POX: The POX Core object`_.


How can I use POX with Mininet?
===============================

Use the :mono:`remote` controller type.  For example, if you are running POX on the same machine as Mininet:

.. code-block:: bash

  mn --topo=linear --mac --controller=remote

(The :mono:`--mac` option is optional, but can make debugging easier.)

A common configuration is to run Mininet in a virtual machine and run POX in your host environment.  In this case, point Mininet at an IP address of the host environment.  If you're using VirtualBox and a "Host-only Adapter", this is the address assigned to the VirtualBox virtual adapter (e.g., :mono:`vboxnet0`).  You do this slightly differently if you're using Mininet 1 or Mininet 2.  For Mininet 1:

.. code-block:: bash

  mn --topo=linear --mac --controller=remote --ip=192.168.56.1

For Mininet 2:

.. code-block:: bash

 mn --topo=linear --mac --controller=remote,ip=192.168.56.1

Additionally, in Mininet 2, you may want to specify the :mono:`--nolistenport` option.


I'm seeing many packet_in messages and forwarding isn't working; what gives?
============================================================================

This problem is often seen when attempting to use one of the "learning" forwarding components (:mono:`l2_learning`, :mono:`l2_multi`, etc.) on a mesh or other topology with a loop.  These forwarding components do not take into account the adjustments that are required to work on loopy topologies.

The :mono:`spanning_tree` component is meant to be a fairly generic solution to this problem, so you might try running it as well.  It can also be helpful to prevent broadcasts until discovery has had time to discover the entire topology.  Some components (such as :mono:`l2_learning`) have an option to enforce this.


Does POX support topologies with loops?
=======================================

Let's start with making it clear that this is a broken question because POX itself simply doesn't care.  The real question is "Do any of the forwarding components that come with POX support topologies with loops?"  The answer is ... sort of!  As far as I remember, none of them *explicitly* support loops.  However, several of them are compatible with the :mono:`openflow.spanning_tree` component.  :mono:`openflow.spanning_tree` disables flooding on ports, leaving only a tree.  For forwarding components which only loop during floods, don't change the port flood bit, and work with the discovery component, this may be good enough.

See the section on the :mono:`openflow.spanning_tree` component and the above FAQ question for more.


Switches keep disconnecting (especially with Pantou/reference switch).  Help?
=============================================================================

The short answer is that you should run the :mono:`openflow.keepalive` component too.  See the description of this component above for more information.


Why doesn't the openflow.webservice component work?
===================================================

If you're sending requests to the :mono:`openflow.webservice` component and it's not sending back replies, this chances are that you're not conforming to the JSON-RPC spec.  Specifically, you're not including an ":mono:`id`" key in your request.  Add one and set it to an integer and see if that helps.  See the :mono:`openflow.webservice` examples in this manual for additional information.


What are these log messages from the packet subsystem?
======================================================

You may see info level log messages from the packet subsystem like:

:mono:`(dhcp parse) warning DHCP packet data too short to parse header`

:mono:`(udp parse) warning UDP packet data too short to parse header: data len X`

:mono:`(icmp parse) warning ICMP packet data too short`

*These aren't errors or necessarily indicative of any problem.*

When OpenFlow switches send packets to the controller, they often do not send the entire packet.  When the packet is sent to the controller via an output action to the OFPP_CONTROLLER port, the output action can contain the number of bytes to send.  When a packet is sent to the controller due to a table miss, this length can be set via OFPT_SET_CONFIG, but often defaults to 128 bytes.

When the switch only sends a partial packet, POX's packet library may well not be able to parse the entire packet since the entire packet isn't actually there.  The decision was made to log this message at info level instead of debug level because debug messages within non-component portions of POX are intended to help debug POX itself, and the condition leading to these isn't indicative of bugs in POX.  *It's possible that we should bend the rules here and log them at debug level anyway; feel free to register your opinion on noxrepo.org (in the forum or pox-dev mailing list).*

Some things you can do about this:

 #. Ignore it unless it's actually causing a problem for you (because your application requires these packets to be parsed correctly).
 #. Turn the packet subsystem's log level to warning (:mono:`log.level --packet=WARN`).
 #. Install a flow to send the entirety of relevant packets to the controller (an ofp_action_output to OFPP_CONTROLLER will default to doing this).
 #. Tell the switch to send complete packets to the controller on table misses (an easy way is to simply invoke the :mono:`misc.full_payload` component).

I Installed IP-Based Table Entries But Ping/TCP Doesn't Work.  Why not?
=======================================================================

To answer your question with a question: are you handling ARP?


Does POX support Python 3?
==========================

Not yet.

At this point, Python 2 is still pretty much the standard. Additionally, it's quite common to run POX using the PyPy interpreter, which does not yet support Python 3.  In particular, we don't have much desire to support *both* Python 2 and 3 simultaneously.  So we expect to someday support Python 3, but not until it seems like it's what the majority of users want, and probably not until after PyPy does.

If Python 3 support is important to you *now*, you should start an issue on the github tracker or post about it on pox-dev.  Especially if you're willing to do some of the work, we'll be happy to discuss getting this done, how we can help, and how we can get your work merged into the main repository.

(This question hasn't actually been asked a single time, much less frequently, as its inclusion in a FAQ would imply.  I just wanted to document the answer.)


I'd like to contribute.  Can I?  Do you have project ideas?
===========================================================

Sure you can.  Some thoughts:

* Read the `Coding Conventions`_ section of the manual
* Join the pox-dev mailing list at noxrepo.org
* Get a github account and fork the POX repository
* Submit pull requests on github or formatted patches on pox-dev

If you're looking for project ideas:

* Check POX's issue tracker on github
* Ask the mailing list – in particular, Murphy has started maintaining a list of projects suitable for students or interested parties and may be able to give you a suggestion


What's this warning like "core:Still waiting on 1 component(s)"?
================================================================

A dependency of some component you're running hasn't been met.  To rectify this, you should probably run the missing component by including it on the commandline.  If you run with logging at DEBUG level, you'll get a more detailed message, such as ":mono:`core:startup() in pox.forwarding.l2_multi still waiting for: openflow_discovery`".  In this case, it indicates that :mono:`forwarding.l2_multi` requires :mono:`openflow.discovery`.


Why doesn't POX's discovery use the normal LLDP MAC address?
============================================================

POX's discovery uses the MAC address that Nicira defined for use with OpenFlow-based discovery and which was used by the NOX controller.  The significant difference between the normal one and the Nicira one is that the original one is in the bridge-filtered range, which means that Ethernet switches should never forward it.  If your network is entirely made of OpenFlow switches, this doesn't make any difference.  But if your network is a combination of OpenFlow switches and traditional Ethernet switches, it does.

Imagine you had the following topology, where A and B are OpenFlow switches, and S is a standard Ethernet switch. 

:mono:`A -- S -- B`

Imagine that the controller tells A to send a discovery packet with the normal LLDP MAC address.  It gets to S.  S will drop it, since the address is bridge-filtered.  Thus, the controller concludes that packets sent from A will not reach B.  This is obviously false except in the very special (that is, unusual) case where you're using bridge-filtered MAC addresses!

By using a non-bridge-filtered address for discovery, POX (and NOX before it) can "see through" traditional Ethernet switches.


What's a good strategy for debugging a problem with my POX-based controller?
============================================================================

The `Open vSwitch FAQ <https://raw.githubusercontent.com/openvswitch/ovs/master/FAQ>`_ has a really good entry on this subject:

  Q: I have a sophisticated network setup involving Open vSwitch, VMs or multiple hosts, and other components. The behavior isn't what I expect. Help!

  A: To debug network behavior problems, trace the path of a packet, hop-by-hop, from its origin in one host to a remote host. If that's correct, then trace the path of the response packet back to the origin. Usually a simple ICMP echo request and reply ("ping") packet is good enough. Start by initiating an ongoing "ping" from the origin host to a remote host. If you are tracking down a connectivity problem, the "ping" will not display any successful output, but packets are still being sent. (In this case the packets being sent are likely ARP rather than ICMP.)

The entry then goes on in further detail, including information on some of the tools available to you.  While they are Open vSwitch-centric, a lot of it applies (perhaps in slightly altered form) to debugging SDN programs in general.


I've got a problem / bug!  Can you help me?
===========================================

Possibly.  There are a number of things you can do to help yourself, though.  If none of those work or apply, there are a number of things you can do to help us help you.  Here are some things you might try:

#. *Read the logs.*  If the logs don't seem to say anything useful, try reading them at a lower log level, such as the DEBUG level.  That way, you'll get *all* the log messages.  Do this by adding :mono:`log.level --DEBUG` to your commandline.  See the :mono:`log.level` component section for more info on adjusting log levels.  In particular, pay attention for warnings or errors (e.g., see the "Still waiting on..." FAQ entry)!
#. *Look at the OpenFlow traffic.*  If you don't seem to be getting an event that you think you should be, or you think you're sending messages but the switch doesn't seem to be responding to, or anything else where you think there's a breakdown in communication between a switch and POX, it can be helpful to look at what actually got put on the wire.  There is an OpenFlow dissector for Wireshark (you can Google for it).  You can either run it as usual, or you can use POX's :mono:`openflow.debug` component to generate synthetic traces which show exactly what POX thinks it saw – one message per synthetic "packet" (which makes the Wireshark list easier to read).
#. *Run a newer version.*  Particularly, if you are running a release branch, you might think about running the active branch instead.  While active branches may contain new problems, they also fix old ones!  See the "Selecting a Branch / Version" section for more information.
#. *Check the other FAQs. * Your question may already be answered!
#. *Search the mailing list archive.*  This would be more helpful if it weren't so flaky.  Sorry about that, hopefully we'll get around to fixing it before too long!

If none of those work, you might try posting to the pox-dev mailing list (sign up at `www.noxrepo.org/community/mailing-lists <http://www.noxrepo.org/community/mailing-lists/>`_).  When you do, you'll probably get better results the more you can do the following:

#. *Post the commandline with which you invoked POX.*
#. *Post the POX log.*  It's probably a good idea to post them at DEBUG level.  Even if you didn't see anything in the log, it may be helpful to someone else.  The first part of the log (before the Up message) is especially useful, as it tells which operating system and Python interpreter you are running, and many components announce themselves.  If you don't post this, you might at least try to include some of this information yourself.
#. *Post the traceback.*  This sort of goes with the prior entry, but it's worth making specific note.  If you get an exception, post the full text of the exception including the traceback if possible.
#. *Post which version of POX you are using.*  Did you just do a :mono:`git clone http://github.org/noxrepo/pox`, or did you switch branches?  Did you do this recently or are you potentially using an older version?
#. *Post what kind of switches you are using.*  Are you running POX with Mininet?  Which version?  What commandline did you use to invoke Mininet?  If it's custom, consider posting your Mininet topology or script.  If you're running with hardware switches, what kind?  If you're running with a software switch, which one and which version?
#. *Post code which illustrates the problem.*  A minimal example is great, but anything is better than nothing.
#. *Post a trace of the controller-switch OpenFlow traffic. * This is data you should have collected yourself as part of step 2 of the previous list.  Capture the traffic with Wireshark or the :mono:`openflow.debug` component and post it to the list.
#. *Post what you've tried already.*  Hopefully you've tried to address the issue yourself.  What have you tried and what were the results?

Doing the above makes it easier for people to help you, and also potentially saves time – if you don't do the things mentioned above, it's quite possible that the first suggestions you get from the mailing list will be to try the things mentioned above!

.. note:: If you're new to mailing lists and asking questions online, you may find some General Mailing List Tips useful. [#]_

.. [#] In general, the POX mailing list is polite and friendly and useful and other good things, and it has not suffered the negativity/flaming/etc. that people sometimes associate with mailing lists (usually larger ones).  That said, if you're new to mailing lists, it's still not a bad idea to read a bit about mailing list etiquette and how to ask questions on a mailing list.  This is not necessarily because it'll help you avoid rudeness or anything, but because *it will make more people want to help you* (which is good for you), and it will allow the ones that do to *get you the answers you want faster* (which is good for everyone). |brk| Eric S. Raymond, who wrote `The Cathedral and the Bazaar: Musings on Linux and Open Source by an Accidental Revolutionary <http://en.wikipedia.org/wiki/The_Cathedral_and_the_Bazaar>`_ (along with NetHack and some other good stuff), has written up a guide on `How To Ask Questions The Smart Way <http://www.catb.org/esr/faqs/smart-questions.html>`_ (on mailing lists).  In particular, the "`Be precise and informative about your problem <http://www.catb.org/esr/faqs/smart-questions.html#beprecise>`_", "`Describe the goal, not the step <http://www.catb.org/esr/faqs/smart-questions.html#goal>`_", and "`When asking about code <http://www.catb.org/esr/faqs/smart-questions.html#code>`_" sections are good reading.  While it has nothing to do with POX at all, the Apache Jena project has a much shorter but still good guide on `How to ask a good question <http://jena.apache.org/help_and_support/>`_.
