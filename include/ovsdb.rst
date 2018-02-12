OVSDB in POX
------------

.. warning:: *Under Construction:* Official but experimental OVSDB support is appearing in POX.  The code is in an early state, and so is this section.  If you're interested in OVSDB in POX and want help, you probably want to turn to the pox-dev mailing list for now.

Open vSwitch is software switch that can be controller via OpenFlow and is common in in virtual server environments and even as an element of hardware switches.  While the forwarding behavior can be controlled by OpenFlow, OVS has many other aspects which are outside the scope of OpenFlow.  The configuration and state for these are held in the Open vSwitch Database (the OVSDB), which can be queried and manipulated via the JSON-based OVSDB protocol.

POX has support for the OVSDB protocol, which allows one to connect to, configure, and query OVS instances.  There is both an object-oriented API, and a Python-embedded domain specific language for doing this.  The former is somewhat awkward to use -- the expectation is that the latter (which is implemented using the former) will be the primary mode of usage, it is the focus of this section.

The POX OVSDB DSL is a fairly straightforward transliteration of the low-level OVSDB wire protocol.  There are two major implications here.  First, it's all fairly low level.  the DSL is not mean to abstractly model the internals and abilities of OVS -- it's just meant to let you communicate with it.  If you want to implement an abstraction, you can do so atop the DSL (and then share your code with the mailing list!).  Secondly, this documentation is not meant to be stand-alone.  Since the DSL is fairly low level, it maps very closely to the protocol itself, and documenting the protocol in detail here would be redundant.  You should read this section along with `The Open vSwitch Database Management Protocol documentation (RFC 7047) <https://tools.ietf.org/html/rfc7047>`_.

Below are listed valid POX OVSDB DSL statements.  They look vaguely like SQL (no huge surprise).  While we show them here with spaces between words, when writing in Python, words are actually separated by pipe symbols.  For example: :mono:`SELECT|"colname"|FROM|"tablename"`.


Transactions
============

.. todo:: Unfinished

Getting data with SELECT
========================

.. code-block:: none

  SELECT [<columns>] FROM <table> WHERE <condition> [AND <condition> ...]

columns can be a list/tuple of columns, or a list of columns separated by AND.


Modifying data: INSERT, UPDATE, DELETE, and MUTATE
==================================================

.. code-block:: none

  INSERT <row> INTO <table> [WITH UUID_NAME <uuid-name>]

.. code-block:: none

  UPDATE <table> [WHERE <conditions>] WITH <row>


.. code-block:: none

  DELETE [IN|FROM] <table> [WHERE <conditions>]

or

.. code-block:: none

  DELETE [WHERE <conditions>] IN|FROM <table>


.. code-block:: none

  IN <table> [WHERE <conditions>] MUTATE <mutations>

    mutations is an AND-separated list of one of:

    <column> INCREMENT/DECREMENT/MULTIPLYBY/DIVIDEBY/REMAINDEROF <value>

.. code-block:: none

  DELETE <value> FROM <column>

.. code-block:: none

  INSERT <value> INTO <column>


Locks
=====

.. code-block:: none

  ASSERT OWN [LOCK] <lock-id>


Miscellaneous
=============

.. code-block:: none

  COMMIT [DURABLE]

.. code-block:: none

  ABORT

.. code-block:: none

  COMMENT [<comment>]


Waiting for conditions and monitoring changes with WAIT and MONITOR
===================================================================

.. code-block:: none

  WAIT UNTIL/WHILE <columns> [WHERE <conditions>] IN <table> ARE|IS [NOT] <rows> [[WITH] TIMEOUT <timeout>]

columns is a list/tuple or AND-separated list of column names

rows in AND-separated list of rows


.. code-block:: none

  MONITOR [<columns>] IN <table> [FOR [INITIAL] [INSERT] [DELETE] [MODIFY]]
