

Invoking POX
------------

.. note:: **Quick Start**: If you just want a quick start, try: |brk| :mono:`./pox.py samples.pretty_log forwarding.l2_learning`

POX is invoked by running :mono:`pox.py` or :mono:` debug-pox.py`.  The former is meant for running under ordinary circumstances.  The latter is meant for when you're trying to debug problems (it's a good idea to use :mono:`debug-pox.py` when doing development).

POX itself has a couple of optional commandline arguments than can be used at the start of the commandline:

=====================================  =========================================================================================
option                                 meaning
|nobrs| :mono:`--verbose` |nobre|      Display extra information (especially useful for debugging startup problems) |br| *Note:* Generally this is not what you want, and what you want is actually to adjust the logging level via the log.level component.
|nobrs| :mono:`--no-cli` |nobre|       Do not start an interactive shell (No longer applies as of betta)
|nobrs| :mono:`--no-openflow` |nobre|  Do not automatically start listening for OpenFlow connections (Less useful starting with dart, which only loads OpenFlow on demand)
=====================================  =========================================================================================

.. todo:: Add --unthreaded-sh to the above

But running POX by itself doesn't do much – POX functionality is provided by *components* (POX comes with a handful of components, but POX's target audience is really people who want to be developing their own).  Components are specified on the commandline following any of the POX options above. An example of a POX component is :mono:`forwarding.l2_learning`.  This component makes OpenFlow switches operate kind of like L2 learning switches.  To run this component, you simply name it on the command line following any POX options:

.. code-block:: bash

 ./pox.py --no-cli forwarding.l2_learning


You can specify multiple components on the command line.  Not all components work well together, but some do.  Indeed, some components depend on other components, so you may *need* to specify multiple components.  For example, you can run POX's web server component along with :mono:`l2_learning`:

.. code-block:: bash

 ./pox.py --no-cli forwarding.l2_learning web.webcore

Some components take arguments themselves.  These follow the component name and (like POX arguments) begin with two dashes.  For example, :mono:`l2_learning` has a "transparent" mode where switches will even forward packets that are usually dropped (such as LLDP messages), and the web server's port number can be changed from the default (8000) to an arbitrary port.  For example:

.. code-block:: bash

 ./pox.py --no-cli forwarding.l2_learning --transparent web.webcore --port=8888

(If you're starting to think that command lines can get a bit long and complex, there's a solution: write a simple component that just launches other components.)


