Components in POX
-----------------

When we talk about components in POX, what we really mean is something that we can put on the POX command line as described in "Invoking POX".  In the following sections, we discuss some of the components that come with POX and how you can go about creating your own.


Stock Components
================

POX comes with a number of stock components.  Some of these provide core functionality, some provide convenient features, and some are just examples.  The following is an *incomplete* list.


py
**

This component causes POX to start an interactive Python interpreter that can be useful for debugging and interactive experimentation.  Before the betta branch, this was the default behavior (unless disabled with the now obsolete :mono:`--no-cli`).  Other components can add functions / values to this interpreter's namespace (see :mono:`proto.arp_responder` for an example).


forwarding.hub
**************

The hub example just installs wildcarded flood rules on every switch, essentially turning them all into $10 ethernet hubs.


forwarding.l2_learning
**********************

This component makes OpenFlow switches act as a type of L2 learning switch.  This one operates much like NOX's "pyswitch" example, although the implementation is quite different.  While this component learns L2 addresses, the flows it installs are exact-matches on as many fields as possible.  For example, different TCP connections will result in different flows being installed.


forwarding.l2_pairs
*******************

Like l2_learning, this component also makes OpenFlow switches act like a type of L2 learning switch.  However, this one is probably just about the simplest possible way to do it correctly.  Unlike l2_learning, l2_pairs installs rules based purely on MAC addresses.


forwarding.l3_learning
**********************

This component is not quite a router, but it's also definitely not an L2 switch.  It's an L3-learning-switchy-thing.  Perhaps the most useful aspect of it is that it serves as a pretty good example of using POX's packet library to examine and construct ARP requests and replies.

:mono:`l3_learning` does not really care about conventional IP stuff like subnets – it just learns where IP addresses are.  Unfortunately, hosts usually *do* care about that stuff.  Specifically, if a host has a gateway set for some subnet, it really wants to communicate with that subnet through that gateway.  To handle this, you can specify "fake gateways" in the commandline to :mono:`l3_learning`, which will make hosts happy.  For example, if you have some machines which think they're on 10.x.x.x and others that think they're on 192.168.0.x and they think there are gateways at the ".1" addresses:


.. code-block:: bash

    ./pox.py forwarding.l3_learning --fakeways=10.0.0.1,192.168.0.1



forwarding.l2_multi
*******************

This component can still be seen as a learning switch, but it has a twist compared to the others.  The other learning switches "learn" on a switch-by-switch basis, making decisions as if each switch only had local information.  l2_multi uses openflow.discovery to learn the topology of the entire network: as soon as one switch learns where a MAC address is, they all do.  Note that this means you must include openflow.discovery on the commandline.


forwarding.l2_nx
****************

A quick-and-dirty learning switch for Open vSwitch – it uses Nicira extensions as found in Open vSwitch.

Run with something like:


.. code-block:: bash

    ./pox.py openflow.nicira --convert-packet-in forwarding.l2_nx


This forwards based on ethernet source and destination addresses.  Where :mono:`l2_pairs` installs rules for each pair of source and destination address, this component uses two tables on the switch -- one for source addresses and one for destination addresses.

Note that unlike the other learning switches *we keep no state in the controller*.  In truth, we could implement this whole thing using OVS's learn action, but doing it something like is done here will still allow us to implement access control or something at the controller.


forwarding.topo_proactive
*************************

Installs forwarding rules based on topologically significant IP addresses.  We also issue those addresses by DHCP.  A host must use the assigned IP!

Most rules are installed proactively.  This component was added in the carp branch.  The routing code is based on forwarding.l2_multi.

Depends on openflow.discovery and at least sort of works with openflow.spanning_tree (not particularly tested or examined).


openflow.spanning_tree
**********************

This component uses the discovery component to build a view of the network topology, constructs a spanning tree, and then disables flooding on switch ports that aren't on the tree.  The result is that topologies with loops no longer turn your network into useless hot packet soup.

Note that this does not have much of a relationship to Spanning Tree Protocol.  They have similar purposes, but this is a rather different way of going about it.

The :mono:`samples.spanning_tree` component demonstrates this module by loading it and one of several forwarding components.

This component has two options which alter the startup behavior:

:mono:`--no-flood` disables flooding on all ports as soon as a switch connects; on some ports, it will be enabled later.

:mono:`--hold-down` prevents altering of flood control until a complete discovery cycle has completed (and thus, all links have had an opportunity to be discovered).

Thus, the safest (and probably the most sensible) invocation is :mono:`openflow.spanning_tree --no-flood --hold-down` .


openflow.webservice
*******************

A simple JSON-RPC-ish web service for interacting with OpenFlow.  It's derived from the :mono:`of_service` messenger service, so see its docs (in the reference/pydoc) for some additional details.

It requires the webcore component.  You access it by sending an HTTP POST to :mono:`http://wherever_webcore_is_running/OF/`.  The POST data is a JSON string containing (at least) a "method" key containing the name of the method to invoke, and a "params" key which contains a dictionary of argument names and their values.


Current methods include:


================ ============================================= ========================================================
method           description                                   arguments
================ ============================================= ========================================================
set_table        Sets the flow table on a switch.              dpid - a string dpid |br| flows - a list of flow entries
get_switch_desc  Gets switch details.                          dpid - a string dpid 
get_flow_stats   Get list of flows in a table.                 dpid - a string dpid |br| match - match structure (optional, defaults to match all) |br| table_id - table for flows (defaults to all) |br| out_port - filter by out port (defaults to all)
get_switches     Get list of switches and their basic info.    None.
================ ============================================= ========================================================


Example: Get list of connected switches
#######################################

This is pretty easy:

.. code-block:: bash

  curl -i -X POST -d '{"method":"get_switches","id":1}' http://127.0.0.1:8000/OF/

Note the use of the ":mono:`id`" field.  This is a requirement of JSON-RPC as per the various specfications.  Without it, the call is interpreted as a notification – for which the server should not return a value.  POX doesn't really care much what you put in this field, though the JSON-RPC specs do say some stuff about it which you would be wise to not entirely ignore.  An integer is a safe bet.

.. note:: If you don't include an "id" key, you will not get a response!  The above paragraph explains why, but it's worth pointing it out again!



Example: Making a hub using the webservice
##########################################

We can turn the switch with DPID 00-00-00-00-00-01 into a hub by inserting a table entry which matches all packets and sends them to the special OFPP_ALL port.



.. code-block:: bash

  curl -i -X POST -d '{"method":"set_table","params":{"dpid":"00-00-00-00-00-01", \
                     "flows":[{"actions":[{"type":"OFPAT_OUTPUT","port":"OFPP_ALL"}], \
                     "match":{`]`' http://127.0.0.1:8000/OF/



web.webcore
***********

The webcore component starts a web server within the POX process.  Other components can interface with it to provide static and dynamic content of their own.


messenger
*********

The messenger component provides an interface for POX to interact with external processes via bidirectional JSON-based messages.  The messenger by itself is really just an API, actual communication is implemented by *transports*.  Currently, transports exist for TCP sockets and for HTTP.  Actual functionality is implemented by *services*.  POX comes with a few services.  messenger.log_service allows for interacting with the log remotely (reading it, reconfiguring it, etc.).  openflow.of_service allows for some OpenFlow operations (e.g., listing switches, setting flow entries, etc.).  There are also a few small example services in the messenger package, and pox-log.py (in the tools directory) is a small, standalone, external Python application which interfaces with the the logging service over TCP.

By writing a new service, it becomes available over any transport.  Similarly, writing a new transport allows for accessing any service in a new way.

The messenger package in the repository has a fair amount of comments.  Additionally, you can see POXDesk (mentioned elsewhere) as an example of both implementing a new service, and communicating with messenger over HTTP from JavaScript.


Getting Started with the Messenger Component
############################################

To get messenger running, run the messenger component along with some transport(s).  For the sake of example, we'll run the passive (listening) TCP transport and the example messenger service:

.. code-block:: none
  :linenos:

  [pox_dart]$ ./pox.py log.level --DEBUG messenger messenger.tcp_transport messenger.example
  POX 0.3.0 (dart) / Copyright 2011-2014 James McCauley, et al.
  DEBUG:boot:Not launching of_01
  DEBUG:core:POX 0.3.0 (dart) going up...
  DEBUG:core:Running on CPython (2.7.5/Sep 12 2013 21:33:34)
  DEBUG:messenger.tcp_transport:Listening on 0.0.0.0:7790
  DEBUG:core:Platform is Darwin-13.1.0-x86_64-i386-64bit
  INFO:core:POX 0.3.0 (dart) is up.

Now we can connect and use services by connecting to the listening messenger socket.  We can demonstrate this with the :mono:`test_client.py` program.  On starting it up, it connects to the default host/port which gets POX, and messenger sends us a welcome:

.. code-block:: none
  :linenos:

  [messenger]$ python test_client.py
  Connecting to 127.0.0.1:7790
  Recv: {
      "cmd": "welcome",
      "CHANNEL": "",
      "session_id": "bQ6QCYI3ICOGLJPOT7HMVXTN7RE"
  }

We can then send a message to one of the bots that the example service set up.  We'll use the "upper" service which just capitalizes messages you send to it.  Line 8 is typed into :mono:`test_client.py` to send a message, and the rest is the reply from POX:

.. code-block:: none
  :linenos:
  :lineno-start: 8

  {"CHANNEL":"upper","msg":"hello world"}
  Recv: {
      "count": 1,
      "msg": "HELLO WORLD",
      "CHANNEL": "upper"
  }




openflow.of_01
**************

This component communicates with OpenFlow 1.0 (wire protocol 0x01) switches.  When other components that use OpenFlow are loaded, this component is usually started with default values automatically.  However, you may want to launch it manually in order to change its options.  You may also want to launch it manually to run it multiple times (e.g., to listen for OpenFlow connections on multiple ports).

====================================== =============== =================================================================
option                                 default         notes
====================================== =============== =================================================================
|nobrs| --port=<X> |nobre|             6633            Specifies the TCP port to listen for connections on
|nobrs| --address=<X> |nobre|          all addresses   Specifies the IP addresses of interfaces to listen on
|nobrs| --private-key=<X> |nobre|      None            Enables SSL mode and specifies a key file
|nobrs| --certificate=<X> |nobre|      None            Enables SSL mode and specifies a certificate file
|nobrs| --ca-cert=<X> |nobre|          None            Enables SSL mode and specifies a certificate to validate switches
====================================== =============== =================================================================


To configure SSL, the Open vSwitch INSTALL.SSL file and the man page for ovs-controller have a lot of useful info, including info on how to generate the appropriate files to be passed for the various arguments of this component.


openflow.discovery
******************

This component sends specially-crafted LLDP messages out of OpenFlow switches so that it can discover the network topology.  It raises events (which you can listen to) when links go up or down.

More specifically, you can listen to :mono:`LinkEvent` events on :mono:`core.openflow_discovery`.  When a link is detected, such an event is raised with the :mono:`.added` attribute set to True.  When a link is detected as having been removed or failed, the :mono:`.removed` attribute is set to True.  :mono:`LinkEvent` also has a .link attribute, which is a :mono:`Link` object, and a :mono:`port_for_dpid(<dpid>)` method (pass it the DPID of one end of the link and it will tell you the port used on that datapath).

:mono:`Link` objects have the following attributes:

================================ =============================================================
name                             value
================================ =============================================================
dpid1                            The DPID of one of the switches involved in the link
port1                            The port on dpid1 involved in the link
dpid2                            The DPID of the other switch involved in the link
port2                            The port on dpid2
uni                              A "unidirectional" version of the link.  This normalizes the order of the DPIDs and ports, allowing you to compare two links (which may be different directions of the same physical links).
|nobrs| end[0 or 1] |nobre|      The ends of the link as a tuple, i.e., end[0] = (dpid1,port1)
================================ =============================================================

A number of the other example components use discovery and can serve as demonstrations for using discovery.  Obvious possibilities are :mono:`misc.gephi_topo` and :mono:`forwarding.l2_multi`.


openflow.debug
**************

Loading this component will cause POX to create pcap traces containing OpenFlow messages, which you can then load into Wireshark to analyze.  All the headers are synthetic so it's not totally a replacement for actually running tcpdump or Wireshark. It does, however, have the nice property that there is exactly one OpenFlow message in each frame (which makes it easier to look at!).


openflow.keepalive
******************

This component causes POX to send periodic echo requests to connected switches.  This addresses two issues.

First, some switches (including the reference switch) will assume that an idle control connection indicates a loss of connectivity to the controller and will disconnect after some period of silence (often not particularly long).  This behavior is almost certainly broken: one can easily argue that if the switches want to disconnect when the connection is idle, it is *their* responsibility to send echo requests, but arguing won't fix the switches.

Secondly, if you lose network connectivity to the switch, you don't immediately get a FIN or a RST, so it's hard to say exactly when you'll notice that you've lost the switch.  By sending echo requests and tracking their responses, you get a bound on how long it will take to notice.

.. todo:: This component is badly named and will probably be renamed in a future version (possibly eel).


proto.pong
**********

The pong component is a sort of silly example which simply watches for ICMP echo requests (pings) and replies to them.  If you run this component, all pings will seem to be successful!  It serves as a simple example of monitoring and sending packets and of working with ICMP.


proto.arp_responder
*******************

An ARP utility that can learn and proxy ARPs, and can also answer queries from a list of static entries.  This component also adds the ARP table to the interactive console as ":mono:`arp`" – allowing you to interactively query and modify it.

Simply specify IP addresses and the ethernet address you want to associate with them as options:


.. code-block:: bash

  proto.arp_responder --192.168.0.1=00:00:00:00:00:01 --192.168.0.2=00:00:00:00:00:02



info.packet_dump
****************

A simple component that dumps packet_in info to the log.  Sort of like running tcpdump on a switch.


proto.dns_spy
*************

This component monitors DNS replies and stores their results.  Other components can examine them by accessing :mono:`core.DNSSpy.ip_to_name[<ip address>]` and :mono:`core.DNSSpy.name_to_ip[<domain name>]`.


proto.dhcp_client
*****************

A DHCP client component.  This is probably not useful on its own, but can be useful in conjunction with other components.


proto.dhcpd
***********

This is a simple DHCP server.  By default, it claims to be 192.168.0.254, and serves clients addresses in the range 192.168.0.1 to 192.168.0.253, claiming itself to be the gateway and the DNS server.

Note: You might want to use :mono:`proto.arp_responder` to make 192.168.0.254 (or whatever you choose as the IP address) ARP-able.

There are a number of options you can configure:

================ =======================================================================================================================================
Option           Meaning
================ =======================================================================================================================================
network          Subnet to allocate addresses from, e.g., "192.168.0.0/24" or "10.0.0.0/255.0.0.0"
first            First'th address in subnet to use (256 is x.x.1.0 in a /16).
last             Last'th address in subnet to use (258 is x.x.1.2 in a /16).  If 'None', use rest of valid range.
count            Alternate way to specify last address to use
ip               IP to use for DHCP server
router           Router IP to tell clients. Defaults to whatever is set for :mono:`ip`. 'None' will stop the server from telling clients anything.
dns              DNS server to tell clients.  Defaults to whatever is set for :mono:`router`. 'None' will stop the server from telling clients anything.
================ =======================================================================================================================================

Example:

.. code-block:: bash

  proto.dhcpd --network=10.1.1.0/24 --ip=10.1.1.1 --first=10 --last=None --router=None --dns=4.2.2.1


You can also launch this component as :mono:`proto.dhcpd:default` to serve 192.168.0.100-199.

Right before issuing an address, the DHCP server raises a DHCPLease event which you can listen to if you want to learn or deny address allocations:

.. code-block:: python

  def _I_hate_00_00_00_00_00_03 (event):
    if event.host_mac == EthAddr("00:00:00:00:00:03"):
      event.nak() # Deny it!

  core.DHCPD.addListenerByName('DHCPLease', _I_hate_00_00_00_00_00_03)



misc.of_tutorial
****************

This component is for use with the `OpenFlow tutorial <http://www.openflow.org/wk/index.php/OpenFlow_Tutorial>`_.  It acts as a simple hub, but can be modified to act like an L2 learning switch.


misc.full_payload
*****************

By default, when a packet misses the table on a switch, the switch may only send part of the packet (the first 128 bytes) to the controller.  This component reconfigures every switch that connects so that it will send the full packet.


misc.mac_blocker
****************

This component is meant to be used alongside some other reactive forwarding applications, such as l2_learning and l2_pairs.  It pops up a Tkinter-based GUI that lets you block MAC addresses. 

It works by wedging its own PacketIn handler in front of the PacketIn handler of the forwarding component.  When it wants to block something, it kills the event by returning EventHalt.  Thus, the forwarding component never sees the packet/event, never sets up a flow, and the traffic just dies.

Thus, it demonstrates Tkinter-based GUIs in POX as well as some slightly-advanced event handling (using higher-priority event handlers to block PacketIns).  See the pox.lib.revent section of this manual for more on working with events, and see the FAQ entry for creating a firewall for another not-entirely-dissimilar example that blocks TCP ports.


misc.nat
********

A component which does Network Address Translation.


misc.ip_loadbalancer
********************

This component (which started in the carp branch) is a simple TCP load balancer.


.. code-block:: bash

  ./pox.py misc.ip_loadbalancer --ip=<Service IP> --servers=<Server1 IP>,<Server2 IP>,... [--dpid=<dpid>]


Give it a :mono:`service_ip` and a list of server IP addresses.  New TCP flows to the service IP will be randomly redirected to one of the server IPs.

Servers are periodically probed to see if they're alive by sending them ARPs.

By default, it will make the first switch that connects into a load balancer and ignore the other switches.  If you have a topology with multiple switches, it probably makes more sense to specify which one should be the load balancer, and this can be done with the :mono:`--dpid` commandline option.  In this case, you probably want the rest of the switches to do something worthwhile (like forward traffic), and you may have to create a component that does this for you.  For example, you might create a simple component which does the same thing as :mono:`forwarding.l2_learning` on all the switches besides the load balancer.  You could do that with a simple component like the following:


.. code-block:: python
  :linenos:
  :caption: ext/selective_switch.py

  """
  More or less just l2_learning except it ignores a particular switch
  """
  from pox.core import core
  from pox.lib.util import str_to_dpid
  from pox.forwarding.l2_learning import LearningSwitch


  def launch (ignore_dpid):
    ignore_dpid = str_to_dpid(ignore_dpid)

    def _handle_ConnectionUp (event):
      if event.dpid != ignore_dpid:
        core.getLogger().info("Connection %s" % (event.connection,))
        LearningSwitch(event.connection, False)

    core.openflow.addListenerByName("ConnectionUp", _handle_ConnectionUp)


misc.gephi_topo
***************

Streams detected topology to Gephi.

.. image:: images/gephi_topo.png

`Gephi <http://gephi.org/>`_ is a pretty awesome open-source, multiplatform graph visualization/manipulation/analysis package. It has a plugin for streaming graphs back and forth between it and something else over a network connection. The :mono:`gephi_topo` component uses this to provide visualization for switches, links, and (optionally) hosts detected by other POX components. There's a `blog post <http://www.noxrepo.org/2013/06/pox-and-gephi-graph-streaming-visualization/>`_ about this component on `noxrepo.org <http://noxrepo.org/>`_.

This component is loosely based on POXDesk's :mono:`tinytopo` module.  It requires :mono:`discovery`, and :mono:`host_tracker` is optional.Example usage:

.. code-block:: bash

  ./pox.py openflow.discovery misc.gephi_topo host_tracker forwarding.l2_learning


In July of 2014, Rizwan Jamil posted `a message on pox-dev <http://www.mail-archive.com/pox-dev@lists.noxrepo.org/msg01418.html>`_ describing explicit steps for getting this up and running in Ubuntu.


log
***

POX uses Python's logging system, and the :mono:`log` module allows you to configure a fair amount of this through the commandline. For example, you can send the log to a file, change the format of log messages to include the date, etc.


Disabling the Console Log
#########################

You can disable POX's "normal" log using:


.. code-block:: bash

  ./pox.py log --no-default



Log Formatting
##############

Please see the documentation on Python's `LogRecord attributes <http://docs.python.org/2/library/logging.html#logrecord-attributes>`_ for details on log formatting.  As a quick example, you can add timestamps to your log as follows:


.. code-block:: bash

  ./pox.py log --format="%(asctime)s: %(message)s"


Or with simpler timestamps:


.. code-block:: bash

  ./pox.py log --format="[%(asctime)s] %(message)s" --datefmt="%H:%M:%S"


See the :mono:`samples.pretty_log` component for another example (and, particularly, for an example that uses POX's color logging extension).


Log Output
##########

Log messages are processed by various handlers which then print the log to the screen, save it to a file, send it over the network, etc.  You can write your own, but Python also comes with quite a few, which are documented in the Python reference for `logging.handlers <http://docs.python.org/2/library/logging.handlers.html>`_. POX lets you configure a lot of Python's built-in handlers from the commandline; you should refer to the Python reference for the arguments, but specifically, POX lets you configure:

========================= =====================================================
Name                      Type
========================= =====================================================
stderr                    StreamHandler for stderr stream
stdout                    StreamHandler for stdout stream
File                      FileHandler for named file
WatchedFile               WatchedFileHandler
RotatingFile              RotatingFileHandler
TimedRotatingFile         TimedRotatingFileHandler
Socket                    SocketHandler - Sends to TCP socket
Datagram                  DatagramHandler - Sends over UDP
SysLog                    SysLogHandler - Outputs to syslog service
HTTP                      HTTPHandler - Outputs to a web server via GET or POST
========================= =====================================================

To use these, simply specify the Name, followed by a comma-separated list of the positional arguments for the handler Type.  For example, FileHandler takes a file name, and optionally an open mode (which defaults to append), so you could use:


.. code-block:: bash

  ./pox.py log --file=pox.log


Or if you wanted to overwrite the file every time:


.. code-block:: bash

  ./pox.py log --file=pox.log,w


You can also use named arguments by prefacing the entry with a :mono:`*` and then using a comma-separated list of key=value pairs.  For example:


.. code-block:: bash

  ./pox.py log --*TimedRotatingFile=filename=foo.log,when=D,backupCount=5



log.color
*********

The log.color module colorizes the log when possible.  This is actually pretty nice, but getting the most out of it takes a bit more configuration – you might want to take a look at samples.pretty_log.

Color logging should work fine out-of-the-box on Mac OS, Linux, and other environments with the real concept of a terminal.  On Windows, you need a colorizer such as `colorama <http://pypi.python.org/pypi/colorama/>`_.


log.level
*********

POX uses Python's logging infrastructure.  Different components each have their own loggers, the name of which is displayed as part of the log message.  Loggers actually form a hierarchy – you might have a "foo" logger with a "bar" sub-logger, which together would be known as "foo.bar".  Additionally, each log message has a "level" associated with it, which corresponds to how important (or severe) the message is.  The log.level component lets you configure which loggers show what level of detail.  The log levels from most to least severe are:

+-----------------+
|:mono:`CRITICAL` |
+-----------------+
|:mono:`ERROR`    |
+-----------------+
|:mono:`WARNING`  |
+-----------------+
|:mono:`INFO`     |
+-----------------+
|:mono:`DEBUG`    |
+-----------------+

POX's default level is :mono:`INFO`.  To set a different default (e.g., a different level for the "root" of the logger hierarchy):


.. code-block:: bash

  ./pox.py log.level --WARNING


If you are trying to debug a problem with OpenFlow connections, however, you may want to turn up the verbosity of OpenFlow-related logs.  You can adjust all OpenFlow-related log messages like so:


.. code-block:: bash

  ./pox.py log.level --WARNING --openflow=DEBUG


If this leaves you with too many DEBUG level messages from openflow.discovery which you are not interested in, you can then turn it down specifically:


.. code-block:: bash

  ./pox.py log.level --WARNING --openflow=DEBUG --openflow.discovery=INFO



samples.pretty_log
******************

This simple module uses log.color and a custom log format to provide nice, functional log output on the console.


tk
**

This component is meant to assist in building Tk-based GUIs in POX, including simple dialog boxes.  It is quite experimental.


host_tracker
************

This component attempts to keep track of hosts in the network – where they are and how they are configured (at least their MAC/IP addresses).  When things change, the component raises a HostEvent.

For an example host_tracker usage, see the misc.gephi_topo component.

In short, host_tracker works by examining packet-in messages, and learning MAC and IP bindings that way.  We then periodically ARP-ping hosts to see if they're still there.  Note that this means it relies on packets coming to the controller, so forwarding must be done fairly reactively (as with forwarding.l2_learning), or you must install additional flow entries to bring packets to the controller.

You can set various timeouts (in seconds) from the commandline. Names and defaults:

=========================== =============== ==================================================
Name                        Default         Meaning
=========================== =============== ==================================================
arpAware                    60*2            Quiet ARP-responding entries are pinged after this
arpSilent                   60*20           This is for quiet entries not known to answer ARP
arpReply                    4               Time to wait for an ARP reply before retrial
timerInterval               5               Seconds between timer routine activations
entryMove                   60              Minimum expected time to move a physical entry
=========================== =============== ==================================================

Good values for testing:

.. code-block:: none

  --arpAware=15 --arpSilent=45 --arpReply=1 --entryMove=4

You can also specify how many ARP pings we try before deciding it failed:

.. code-block:: none

  --pingLim=2


datapaths.pcap_switch
*********************

This component implements the switch side of OpenFlow – making an OpenFlow switch which can connect to an OpenFlow controller (which could be the same instance of POX, a different instance of POX, or some other OpenFlow controller altogether!) and forward packets.

It is based on a somewhat more abstract superclass which can be used to implement the switch side of OpenFlow without forwarding packets – e.g., to provide a "virtual" OpenFlow switch (along the lines of FlowVisor), to provide just the OpenFlow interface on top of some other forwarding mechanism (e.g., Click), etc.  This is also useful for prototyping OpenFlow extensions or for debugging (it's relatively easy to modify it to simulate conditions which trigger bugs in controllers, for example).  It is *not* meant to be a production switch – the performance is not particularly good!


Developing your own Components
==============================

This section tries to get you started developing your own components for POX.  In some cases, you might find that an existing component does almost what you want.  In these cases, you might start by making a copy of that component and working from there.


The "ext" directory
*******************

As discussed, POX components are really just Python modules.  You can put your Python code wherever you like, as long as POX can find it (e.g., it's in the :mono:`PYTHONPATH` environment variable).  One of the top-level directories in POX is called "ext".  This "ext" directory is a convenient place to build your own components, as POX automatically adds it to the Python search path (that is, looks inside it for additional modules), and it is excluded from the POX git repository (meaning you can easily check out your own repositories into the ext directory).

Thus, one common way to start building your own POX module is simply to copy an existing module (e.g., :mono:`forwarding/l2_learning.py`) into the :mono:`ext` directory (e.g., :mono:`ext/my_component.py`).  You can then modify the new file and invoke POX as :mono:`./pox.py my_component`.


The launch function
*******************

While naming a loadable Python module on the commandline is enough to get POX to load it, a proper POX component should contain a launch function.  In the generic sense, a launch function is a function that POX calls to tell the component to initialize itself.  This is usually a function actually named :mono:`launch`, though there are exceptions.  The launch function is how commandline arguments are actually passed to the component.


A Simple Example
################

The POX commandline, as mentioned above, contains the modules you want to load.  After each module name is an optional set of parameters that go with the module.  For example, you might have a commandline like:


.. code-block:: bash

  ./pox.py foo --bar=3 --baz --spam=disabled


Since the module name is foo, we have either a directory called :mono:`foo` somewhere that POX can find it that contains an :mono:`__init__.py`, or we simply have a :mono:`foo.py` somewhere that POX can find it (e.g., in the :mono:`ext` directory).  At the bare minimum, it might look like this:

.. code-block:: python

  def launch (bar, baz = "eggs", spam = True):
    print "foo:", bar, baz, spam

Note that :mono:`bar` has no default value, which makes the :mono:`bar` parameter not optional.  Attempting to run :mono:`./pox.py foo` with no arguments will complain about the lack of a value for bar.  Notice that in the example given, bar receives the *string* value "3".  In fact, all arguments come to you as strings – if you want them as some other type, it is your responsibility to convert them.

The one exception to the "all arguments are strings" rule is illustrated with the :mono:`baz` argument.  It's specified on the commandline, but not given a value.  So what does :mono:`baz` actually receive in the :mono:`launch()` function?  Simple: It receives the Python :mono:`True` value.  (If it hadn't been specified on the commandline, of course, it would have received the string "eggs".)

Note that the :mono:`spam` value defaults to :mono:`True`.  What if we wanted to send it a false value – how would we do that?  We could try :mono:`--spam=False`, but that would just get us the string "False" (which if we tested for truthiness is actually Truthy!).  And if we just did :mono:`--spam`, that would get us :mono:`True`, which isn't what we want at all. This is one of those cases where you have to explicitly convert the value from a string to whatever type you actually want.  To convert to an integer or a floating point value, you could simply use Python's built-in :mono:`int()` or :mono:`float()`.  For booleans, you could write your own code, but you might consider :mono:`pox.lib.util`'s :mono:`str_to_bool()` function which is pretty liberal about accepting things like "on" or "true" or "enabled" as meaning :mono:`True`, and sees everything else as :mono:`False`.


Multiple Invocation
###################

Now what if we were to try the following commandline?


.. code-block:: bash

  ./pox.py foo --bar=1 foo --bar=2


You might expect to see:


.. code-block:: bash

  foo: 1 eggs True
  foo: 2 eggs True

Instead, however, you get an exception.  POX, by default, *only allows components to be invoked once*.  However, a simple change to your :mono:`launch()` function allows multiple-invocation:

.. code-block:: python

  def launch (bar, baz = "eggs", spam = True, __INSTANCE__ = None):
    print "foo:", bar, baz, spam

If you try the above commandline again, this time it will work.  Adding the :mono:`__INSTANCE__` parameter both flags the function as being multiply-invokable, and also gets passed some information that can be useful for some modules that are invoked multiple times.  Specifically, it's a tuple containing:

* The number of this instance (0...n-1)
* The total number of instances for this module
* :mono:`True` if this is the last instance, :mono:`False` otherwise (just a comparison between the previous two, but it's handy)

You might, for example, only want your component to do some of its initialization once, even if your component is specified multiple times.  You can easily do this by only doing that part of your initialization if the last value in the tuple is :mono:`True`.

You might also wish to examine the minimal component given in section "OpenFlow Events: Responding to Switches".  And, of course, check out the code for POX's existing components.

.. todo:: Someone should write a lot more about developing components.

