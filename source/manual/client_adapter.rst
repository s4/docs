.. index:: client, adapter

==================
S4 Client Adapter
==================

High-Level Design
------------------

S4 provides a client adapter that allows third-party clients to send events
to and receive events from an S4 cluster. The adapter implements a JSON-based
API, for which client drivers may be implemented in a large variety of
languages.

This document describes the design of the Adapter, provides reference for
implementing a client driver, and describes how to configure S4 with the
adapter.

.. image:: /../_static/adapter.png

The adapter allows events to be injected into an S4 cluster, and can receive
events from an S4 cluster via the Communicaiton Layer. The configuration of the
S4 cluster defines the events that will be dispatched to the adapter. By
default, no events are sent to the adapter cluster. Application developers must
configure their application to send relevant events to the adapters. The
*Configuration* section describes how to do this.


A collection of *stubs*
can be registered with an adapter. These stubs implement one or both of the
interfaces: ``InputStub`` and ``OutputStub``.

One particular instance of a stub is the ``ClientStub`` which implements the
``InputStub`` and ``OutputStub`` interfaces. This is an abstract class that
implements functionality to manage client connections via TCP/IP socket.  A
subclass called ``GenericJsonClientStub`` implements the logic to translate
S4-compatible events to and from JSON, which clients use. Other subclasses of
ClientStub may be implemented to provide more efficient encoding of specific
event types (at the cost of generality).

Clients can perform the following actions:

1. ``send`` events to the S4 cluster. These events may either be keyed or
   keyless. In the latter case, the corresponding events are dispatched
   round-robin.
2. ``receive`` *public* events. I.e. events that any client can receive.
3. send ``Request`` events and receive a set of ``Response`` events. The
   responses are *private* events, addressed to a particular client.

The remainder of this document describes the API that clients must implement to
communicate with the ``GenericJsonClientStub``: the communication protocol and
the data format.

Communication Protocol
-----------------------

Overview
^^^^^^^^

The client adapter stub listens on a known TCP/IP port.  Clients establish a
connection to the adapter, and are issued a universally unique identifier (UUID)
for the connection. While connected, the client can send and receive events.
Upon completion, the client disconnects from the adapter. In case the connection
is interrupted for some reason, the client can reconnect with the same
identifier and resume its session.  

See the "Data Format" section for details about the low-level formatting of the
data.

Establishing Connection
^^^^^^^^^^^^^^^^^^^^^^^

Establishment of a connection to the adapter is decomposed into two steps.
During the first step, *initialization*, the client is given a UUID and a
descriptor of the protocol being used by the adapter. If the client is
compatible with the protocol version used by the adapter, it may proceed to the
next step: *connection*. Upon completion of this step, the client is ready to
send and receive events.

An instance of a client is expected to initialize exactly once, but may connect
more than once (e.g. after disconnection due to a network failure).

Initialization
""""""""""""""
1. Client connects on TCP/IP port and sends an empty byte array.
2. Adapter responds with JSON-encoded map having fields ``uuid`` and
   ``protocol``. Example:

   .. code-block:: javascript

     {
       "uuid": "a54277cb-46ff-4e8b-8580-d3ec8ba2fe1b",
       "protocol": {
                     "name":"generic-json",
                     "versionMajor":1,
                     "versionMinor":0
                   }
     }

3. The server closes the TCP/IP connection.
4. The client must verify that the protocol version is compatible. If it is
   compatible, initializatin is successful.
   

Connection
""""""""""

After successfully initializing, the client connects as follows:

1. Client connects to TCP/IP socket and sends a JSON-encoded map with the
   following keys.

   ``uuid`` *(required)*
       UUID received during initialization as value.
   ``readMode`` *(optional)*
       Types of events that this client expects to receive. Values:

         - ``all`` *(default)*: private events addressed to this client , and
           all public events
         - ``select``: all events from a certain subset of streams. This
           selection is specified using one of the following additional keys:
             * ``readInclude``: array of stream names that should be included.
               All other stream names are excluded.
             * ``readExclude``: array of stream names that should be excluded.
               All other stream names are included.
           One of these keys must be specified. Otherwise, the client's
           connection attempt fails. Specifying both the keys is not explicitly
           prohibited, but is considered bad form.
         - ``private``: only events addressed directly to this client
         - ``none``: no events

   ``writeMode`` *(optional)*
       Whether or not this client will send events into the S4 cluster.
       Values:

         - ``enabled`` *(default)*: client may send events
         - ``disabled``: client will not send events
   
   Example:

   .. code-block:: javascript

     {
       "uuid": "a54277cb-46ff-4e8b-8580-d3ec8ba2fe1b",
       "readMode": "private",
       "writeMode": "enabled"
     }


2. The adapter validates this request.

   - If the adapter accepts the connection, it responds with a success message
     and keeps the connection open:

     .. code-block:: javascript

       { "status": "ok" }


   - If the adapter decides to decline the connection request, it responds with
     a JSON-encoded map with key ``status`` with value ``failed`` and an
     optional ``reason`` field with a reason string. It then closes the TCP/IP
     socket. Example:

     .. code-block:: javascript

       {
         "status": "failed",
         "reason": "unknown readMode public"
       }


Disconnecting
^^^^^^^^^^^^^

Write-enabed clients are expected to disconnect gracefully by sending an empty
byte-array to the adapter. The adapter will react by closing the TCP/IP socket.

Read-only clients (``writeMode: disabled``) can disconnect from the adapter by
closing the TCP/IP socket.

Sending Events into S4 Cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Clients can inject events into an S4 cluster on arbitrarily named streams. The
Java class corresponding to the event object must be in the classpath of the S4
cluster and the adapter.

The request to inject an event is sent as a JSON-encoded map containing the
following keys.

  ``stream`` (required)
      Name of stream on which this event is to be dispatched into S4 cluster.
  ``keys`` (optional)
      Array of strings representing the fields in the event object which should
      be composed to produce the key used for routing the event. If the ``key``
      field is not specified, the event is typically routed round-robin.
  ``class`` (required)
      Java class name of the event object. Use the ``X$Y`` notation to denote a
      class ``Y`` which is nested in ``X``.
  ``object`` (required)
      JSON-encoded event object. The adapter uses `Gson
      <http://http://code.google.com/p/google-gson/>`_ to convert this
      JSON-string into a Java object of type specified in the ``class`` field.

Example:

.. code-block:: javascript

  {
    "stream":"RawSentence",
    "class":"io.s4.example.speech01.Sentence",
    "object": "{\"id\":14000049,\"speechId\":14000000,\"text\":\"We must act quickly.\",\"time\":1242800008000}"
  }

Notice that the ``object`` value is an escaped string. It is a JSON-encoded
string representation of the object within a JSON structure.


Receiving Events from the S4 Cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If the client specified a ``readMode`` other than ``none``, the adapter sends
events to the client. Clients are expected to promptly read events from their
TCP/IP sockets. Failure to do so may result in their connection being terminated
by the adapter.

Each event is a JSON-encoded map containing the following keys and values.

  ``stream``
      Name of stream on which the adapter received the event.
  ``class``
      Java class name of the event object.
  ``object``
      JSON-encoded event object. The adapter uses `Gson
      <http://http://code.google.com/p/google-gson/>`_ to convert the received
      event object into a string.

Example:

.. code-block:: javascript

   {
     "stream":"SentenceJoined",
     "class":"io.s4.example.speech01.Sentence",
     "object":"{\"id\":24000086,\"speechId\":24000000,\"text\":\"What kind of logic is that?\",\"time\":1242801726000,\"location\":\"cleveland, oh, us\"}"}
   }

Implementers of client libraries are encouraged to provide the following:
- a *timed read* procedure that respect a timeout
- a *batch read* procedure that returns all events that arrive within a
  specified time interval.

Data Transmission Format
------------------------

Data transmission between the client and the adapter is in the form of byte
arrays. Strings are first converted into an array of ``byte``-s.

A byte array, ``B``, is sent over a socket as follows:

  ``length`` (4 bytes)
        ``B.length`` encoded as a 32-bit `big endian
        <http://en.wikipedia.org/wiki/Endianness#Big-endian>`_
        integer.
  ``content`` (B.length bytes)
        Bytes of B.


Configuration
-------------

Cluster Configuration
^^^^^^^^^^^^^^^^^^^^^

The S4 and adapter clusters are defined in the ``clusters.xml`` file. Here, the
two clusters are given names. Typically, the S4 compute cluster is called
``s4`` and the adapter cluster is called ``adapter``. Every node in each cluster
must have a partition id that is unique within the cluster and in the range [0,
N-1] where N is the number of nodes in the cluster.

Adapter Configuration
^^^^^^^^^^^^^^^^^^^^^

The adapter Main program scans all application-specific client adapter
configuration files and instantiates all beans of type ``InputStub`` and
``OutputStub``. A particular case of this is the ``GenericJsonClientStub``.

Typically, this is the only configuration that is required (``client_stub.xml``):

.. code-block:: xml

  <bean id="genericStub" class="io.s4.client.GenericJsonClientStub" init-method="init">
    <property name="connectionPort" value="2334"/>
  </bean>


Dipatcher: S4 to Adapter
^^^^^^^^^^^^^^^^^^^^^^^^

A basic component for sending events *from* the S4 cluster *to* the adapter
cluster is a ``CommLayerEmitter``. It is configured by setting the
``listenerAppConfig`` property to reflect the name of the adapter cluster  as
follows in the S4 configuration.

.. code-block:: xml

  <bean id="commLayerEmitterToAdapter" class="io.s4.emitter.CommLayerEmitter" init-method="init">
    <property name="serDeser" ref="serDeser"/>
    <property name="listener" ref="rawListener"/>
    <property name="listenerAppName" value="adapter"/>
    <property name="monitor" ref="monitor"/>
  </bean>

S4 application developers compose PEs in their application configuration to
perform computations and emit events. The destinations for the events may be
other PEs, or may be the client adapter (or both).

In order to allow such configuration of event routing, developers have at their
disposal the following classes (all implement the ``EventDispatcher`` interface):

===============================================   ==============================================================================
EventDispatcher Class                             Description
===============================================   ==============================================================================
``io.s4.dispatcher.Dispatcher``                   Accepts an event object, along with a stream name and an optional set of key
                                                  names. It then determines which node the event should be dispatched to
                                                  (based on the key value) using a ``Partitioner``, and emits the event to
                                                  that partition, typically using ``io.s4.emitter.CommLayerEmitter``.

``io.s4.dispatcher.StreamSelectingDispatcher``    Uses a (configurable) list of stream names to select events: an event is
                                                  selected only when the stream name is present in this list. Selected events
                                                  are delegated to a (configurable) ``EventDispatcher`` for handling.

``io.s4.dispatcher.StreamExcludingDispatcher``    Uses a (configurable) list of stream names to exclude events: an event is
                                                  selected only when the stream name is absent from this list. Selected events
                                                  are delegated to a (configurable) ``EventDispatcher`` for handling.

``io.s4.dispatcher.MultiDispatcher``              Configurable with a set of ``EventDispatcher``-s. Every event is delegated
                                                  to all the member ``EventDispatcher``-s.
===============================================   ==============================================================================


Example
"""""""

In ``speech02`` example application, suppose that the ``SentenceJoined`` stream
has to be sent to the client adapters, in addition to being sent to the *event
catcher*.

We add a new dispatcher which can fork ``SentenceJoined`` events to two
dispatchers: 

.. code-block:: xml

  <!-- Fan out events -->
  <bean id="forkdispatcher" class="io.s4.dispatcher.MultiDispatcher">
    <property name="dispatchers">
      <list>

        <!-- send everything through the standard S4 dispatcher -->
        <ref bean="dispatcher"/>

        <!-- send some streams to client adapters -->
        <bean id="selectiveDispatchToAdapter" class="io.s4.dispatcher.StreamSelectingDispatcher">
          <property name="dispatcher" ref="dispatcherToClientAdapters"/>
          <property name="streams">
            <list>
              <value>SentenceJoined</value>
            </list>
          </property>
        </bean>

      </list>
    </property>
  </bean>

Here, ``dispatcherToClientAdapters`` is configured to only dispatch events on
the ``SentenceJoined`` stream. However, the events need to be sent to *every*
adapter in the cluster since clients may connect to any one of them.

The typical ``s4_conf.xml`` file includes a dispatcher to send events to all
adapter nodes, using the ``BroadcastPartitioner``.

.. code-block:: xml
  <!-- Dispatcher to send to all adapter nodes. -->
  <bean id="dispatcherToClientAdapters" class="io.s4.dispatcher.Dispatcher" init-method="init">
    <property name="partitioners">
      <list>
        <ref bean="broadcastPartitioner"/>
      </list>
    </property>
    <property name="eventEmitter" ref="commLayerEmitterToAdapter"/>
    <property name="loggerName" value="s4"/>
  </bean>

  <!-- Partitioner to achieve broadcast -->
  <bean id="broadcastPartitioner" class="io.s4.dispatcher.partitioner.BroadcastPartitioner"/>


Guidelines for Configuring a S4/Adapter Cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Some considerations must be made before configuring S4 and the client adapter.

1. The subset of events that are sent to the adapter.
"""""""""""""""""""""""""""""""""""""""""""""""""""""

It is recommended that the events dispatched to the adapter be kept to a
minimum. I.e. do not dispatch events to the adapter by default; do so only if
required.

Currently, changing dispatching requires the S4 cluster to be restarted. So care
must be taken to make this decision.

2. The number of adapter nodes and clusters to be used.
"""""""""""""""""""""""""""""""""""""""""""""""""""""""

It is possible to have multiple adapter nodes in an adapter clusters. It is also
possible to have multiple adapter clusters.

In general:

- increase the number of nodes in an adapter cluster if a large number
  of clients are expected to connect.
- increase the number of adapter clusters if a large number of data streams are
  to be sent to the adapters from S4. In that case, dedicate each adapter
  cluster to a subset of these streams.

3. Partitioning of events across adapter nodes
""""""""""""""""""""""""""""""""""""""""""""""

If a single stream has a high volume of events such that no single adapter node
can handle it entirely, consider partitioning the stream across the adapter
cluster.

In that case, the PEs dispatching events on that stream must not use the
``BroadcastPartitioner``. They should instead use the standard
``DefaultPartitioner``. With this, it becomes the burden of the client to
connect to *all* nodes in the adapter cluster in order to receive the entire
event stream.
