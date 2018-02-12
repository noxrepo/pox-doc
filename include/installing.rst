
Installing POX
--------------

.. note:: One of POX's design points is to be easy to install and run.  However, especially if this is all very new to you, there are times when you really want to just be able to dive straight in to a working configuration.  If this describes you, you may want to skip installing POX altogether and instead download a virtual machine image with POX and software OpenFlow switches preinstalled and ready to go.  You can totally do this!  Usually the switches are provided by a tool known as Mininet, which allows for complex software OpenFlow networks to be run on a single machine – or within a single virtual machine. |brk| The official Mininet VMs come with POX installed (and can be easily upgraded to the latest version), so they are certainly one option.  This option is the one assumed by the `Stanford OpenFlow Tutorial <http://archive.openflow.org/wk/index.php/OpenFlow_Tutorial>`_, which you may want to follow if you're looking for a crash course on OpenFlow and POX. |brk| Another option is the `SDNHub POX Tutorial <http://sdnhub.org/tutorials/pox/>`_ which has its own preconfigured VM.

Requirements
============

POX requires Python 2.7.  In practice, it also mostly runs with Python 2.6, and there have been a few commits around March 2013 to improve this somewhat, but nobody is presently *really* trying to support this.  See the FAQ entry `Does POX Support Python 3?`_ for the story on Python 3 support.  If all you have is Python 2.6, you might want to look into PyPy (see below) or `pythonbrew <https://github.com/utahta/pythonbrew/blob/master/README.rst>`_.

POX officially supports Windows, Mac OS, and Linux (though it has been used on other systems as well).  A lot of the development happens on Mac OS, so it almost always works on Mac OS.  Occasionally things will break for the other OSes; the time it takes to fix such problems is largely a function of how quickly problems are reported.  In general, problems are noticed on Linux fairly quickly (especially for big problems) and noticed on Windows rather slowly.  If you notice something not working or that seems strange, please submit an issue on the github tracker or send a message to the pox-dev mailing list so that it can be fixed!

POX can be used with the "standard" Python interpreter (CPython), but also supports `PyPy <http://pypy.org/>`_ (see below).


Getting the Code / Installing POX
=================================

The best way to work with POX is as a `git <http://git-scm.com/>`_ repository.  You can also grab it as a tarball or zipball, but source control is generally a good thing (if you do want to do this, take a look at the `Versions / Downloads page <http://www.noxrepo.org/pox/versionsdownloads/>`_ at NOXRepo.org).

POX is hosted on `github <http://github.com/>`_.  If you intend to make modifications to POX itself, you might consider `making your own github fork <https://help.github.com/articles/fork-a-repo>`_ of it from the `POX repository page <http://github.com/noxrepo/pox>`_.  If you just want to grab it quickly to run or play around with, you can simply create a local clone:

.. code-block:: bash

 $ git clone http://github.com/noxrepo/pox
 $ cd pox


Selecting a Branch / Version
============================

h1. The POX repository has multiple branches.  Specifically, it has at least some *release* branches and at least one *active* branch.  The default branch (what you get if you just do the above commands) will be the most recent release branch.  Release branches may get minor updates, but are no longer being actively developed.  On the other hand, active branches *are* being actively developed.  In theory, release branches should be somewhat more stable (by which we mean that if you have something working, we aren't going to break it).  On the other hand, active branches will contain improvements (bug fixes, new features, etc.).  Whether you should base your work on one or the other depends on your needs.  One thing that may factor into your decision is that you'll probably get better support on the mailing list if you're using an active branch (lots of answers start with "upgrade to the active branch").

The main POX branches are named alphabetically after fish.  You can see the one you're currently on with :mono:`git branch`.  As of this writing, the branches and the approximate dates during which they were undergoing active development are:

* *angler (June 2011 - March 2013)*
* *betta (Until May 2013)*
* *carp (Until October 2013)*
* *dart (Until July 2014)*
* *eel (...)*

This means that (as of this writing!), *dart* is the most recent release branch, and *eel* is the current active branch.  (Sidenote, *angler* is the same as the original *master* branch, which has been removed to avoid confusion.)

To use the *dart* branch, for example, you simply check it out after cloning the repository:

.. code-block:: bash

 ~$ git clone http://github.com/noxrepo/pox
 ~$ cd pox
 ~/pox$ git checkout dart

You can find more information on POX versions / branches `on the main NOXRepo site <http://www.noxrepo.org/pox/versionsdownloads/>`_.


PyPy Support
============

While it's not as heavily tested as the normal Python interpreter, it's a goal of POX to run well on the `PyPy <http://pypy.org/>`_ Python runtime.  There are two advantages of this.  First, PyPy is generally quite a bit faster than CPython.  Secondly, it's very easily portable -- you can easily package up POX and PyPy in a single tarball and have them ready to run.

You can, of course, download, install, and invoke PyPy in the usual way.  On Mac OS and Linux, however, POX also supports a really simple method: Download the latest PyPy tarball for your OS, and decompress it into a folder named :mono:`pypy` alongside :mono:`pox.py`.  Then just run :mono:`pox.py` as usual (:mono:`./pox.py`), and it should use PyPy instead of CPython.
