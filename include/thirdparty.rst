Third-Party Tools, Tutorials, Etc.
----------------------------------

This section attempts to note some projects which use POX but are not part of POX itself.  (This may get moved to its own page or something in the future.)


POXDesk: A POX Web GUI
======================

.. image:: images/POXDesk4.png

This is a side-project of Murphy's in a very early state.  It provides a number of features: a flow table inspector, a log viewer, a simple topology viewer, a terminal, an L2 learning switch implemented in JavaScript, etc.  It's meant to be extensible.  It's implemented using the Qooxdoo JavaScript framework on the front end, and POX's web server and messenger service on the backend.

Site: `github.com/MurphyMc/poxdesk/wiki <https://github.com/MurphyMc/poxdesk/wiki>`_

Blog post with more info and screenshots: `www.noxrepo.org/2012/09/pox-web-interfaces/ <http://www.noxrepo.org/2012/09/pox-web-interfaces/>`_


OpenFlow Tutorial
=================

The OpenFlow Tutorial has a POX version which guides the reader through setting up a test environment using Mininet and implementing a hub and learning switch, among other things.

Site: `www.openflow.org/wk/index.php/OpenFlow_Tutorial <http://www.openflow.org/wk/index.php/OpenFlow_Tutorial>`_


SDNHub POX Controller Tutorial
==============================

SDNHub has a brief tutorial on POX which includes their own VM with POX and Mininet preinstalled.

Site: `sdnhub.org/tutorials/pox/ <http://sdnhub.org/tutorials/pox/>`_


OpenFlow Switch Tutorial
========================

These are examinations of some different ways to write "switch" type applications in OpenFlow with POX.  Prepared by William Emmanuel Yu.

* *OpenFlow Switch Tutorial* (`of_sw_tutorial.py <https://github.com/hip2b2/poxstuff/blob/master/of_sw_tutorial.py>`_) - this is a simple OpenFlow module with various switch implementations. The following implementations are present:

 * Dumb Hub - in this implementation, all packets are sent to the controller and then broadcast to all ports. No flows are installed.
 * Pair Hub - Flows are installed for source and destination MAC address pairs that instruct packets to be broadcast.
 * Lazy Hub - A single flow is installed to broadcast packet to all ports for any packet.
 * Bad Switch - Here flows are installed only based on destination MAC addresses. Find out what the problem is!
 * Pair Switch - Simple to a pair hub in that it installs flows based on source and destination MAC addresses but it forwards packets to the appropriate port instead of doing a broadcast.
 * Ideal Pair Switch - Improvement of the above switch where both to source and to destination MAC address flows are installed. Why is this an improvement from the above switch?

* *OpenFlow Switch Interactive Tutorial* (`of_sw_tutorial_oo.py <https://github.com/hip2b2/poxstuff/blob/master/of_sw_tutorial_oo.py>`_) - this is the same module above that is Interactive. If run with a pox py command line. Users can dynamically load and unload various switch implementations.  *Note:* To use the interactive features, you will also need to include the "py" component on your commandline.

.. code-block:: none
  :caption: Sample Interactive Switch Session

  POX> INFO:openflow.of_01:[00-00-00-00-00-01 1] connected
  POX> MySwitch.list_available_listeners()
  INFO:samples.of_sw_tutorial_oo:SW_BADSWITCH
  INFO:samples.of_sw_tutorial_oo:SW_LAZYHUB
  INFO:samples.of_sw_tutorial_oo:SW_PAIRSWITCH
  INFO:samples.of_sw_tutorial_oo:SW_IDEALPAIRSWITCH
  INFO:samples.of_sw_tutorial_oo:SW_DUMBHUB
  INFO:samples.of_sw_tutorial_oo:SW_PAIRHUB
  POX> MySwitch.clear_all_flows()
  DEBUG:samples.of_sw_tutorial_oo:Clearing all flows from 00-00-00-00-00-01.
  POX> MySwitch.detach_packetin_listener()
  DEBUG:samples.of_sw_tutorial_oo:Detaching switch SW_IDEALPAIRSWITCH.
  POX> MySwitch.attach_packetin_listener('SW_LAZYHUB')
  DEBUG:samples.of_sw_tutorial_oo:Attach switch SW_LAZYHUB.
  POX> MySwitch.clear_all_flows()
  DEBUG:samples.of_sw_tutorial_oo:Clearing all flows from 00-00-00-00-00-01.
  POX> MySwitch.detach_packetin_listener()
  DEBUG:samples.of_sw_tutorial_oo:Detaching switch SW_LAZYHUB.
  POX> MySwitch.attach_packetin_listener('SW_BADSWITCH')
  DEBUG:samples.of_sw_tutorial_oo:Attach switch SW_BADSWITCH.

* *OpenFlow Switch Tutorial for Betta (and beyond)* (`of_sw_tutorial_resend.py <https://github.com/hip2b2/poxstuff/blob/master/of_sw_tutorial_resend.py>`_) - this is the same module as above but takes advantage of resend functionality in the betta branch.

Statistics Collector Example
============================

Prepared by William Emmanuel Yu.

* *Statistics Collector Example* (`flow_stats.py <https://github.com/hip2b2/poxstuff/blob/master/flow_stats.py>`_) - this module collects statistics every 5 seconds using a timer. There are three (3) kinds of statistics collected: port statistics, flow statistics and a sample displaying on web statistics (similar to an example above).  Requires at least the betta version of POX.

RipL: Datacenter Topologies with POX and Mininet
================================================

RipL (Ripcord-Lite) is a Python library by Brandon Heller to facilitate working with datacenter-like networking stuff.  In particular, it makes it really easy to run a "fat tree" topology in Mininet.

RipL-POX builds on RipL, creating an OpenFlow controller for static RipL topologies.  It can route along a simple spanning tree, or it can utilize multiple paths either at random or based on a hash, and it can do these things either proactively or reactively (or as a hybrid).

Murphy has recently (spring 2014) updated RipL-POX to work on recent versions of POX (the current head of the dart branch).  As of this writing, his changes haven't been merged upstream, but are available in his own git repository.  Unfortunately, RipL itself (not RipL-POX) seems to have some incompatibility with current versions of Mininet which prevents the proactive mode of RipL-POX from working.  Murphy has some brief notes on how he cobbled together a working version of Mininet/RipL for testing in a `pull request on the main RipL-POX repository <https://github.com/brandonheller/riplpox/pull/5>`_.

The RipL and RipL-POX repositories have README and INSTALL documents useful for getting going.  Feel free to ask questions in the usual places for additional assistance.


Main RipL repository: `github.com/brandonheller/ripl <https://github.com/brandonheller/ripl>`_

Main RipL-POX repository: `github.com/brandonheller/riplpox <https://github.com/brandonheller/riplpox>`_

Murphy's updated RipL-POX repository: `github.com/MurphyMc/riplpox <https://github.com/MurphyMc/riplpox>`_


Direct Server Return Load Balancer
==================================

David A Dunn has `made available <http://www.noxrepo.org/forum/topic/simple-l2dsr-load-balancer-built-with-pox/>`_ a component which does load balancing for VIP traffic utilizing DSR with other traffic being handled by the "NORMAL" action.

