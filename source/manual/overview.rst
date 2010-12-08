What is S4?
===========

S4 is a general-purpose, distributed, scalable, partially fault-tolerant, pluggable platform that allows programmers to easily develop applications for processing continuous unbounded streams of data. S4 was initially developed to personalize search advertising products at `Yahoo\! <http://yahoo.com/>`_, which operate at a rate of thousands of events per second. MapReduce [1]_ excels at batch jobs, but is hard to apply to stream computation tasks.

In S4, keyed data events are routed with affinity to Processing Elements (PEs), which consume the events and do one or both of the following

* emit one or more events which may be consumed by other PEs,
* publish results, possibly to an external data store or consumer. 

The architecture resembles the Actors [2]_ model, providing semantics of encapsulation and location transparency, thus allowing applications to be massively concurrent while exposing a simple programming interface to application developers. This design choice also makes it relatively easy to reason about correctness due to the general absence of side-effects.

Key Features
------------

**Scalable**
  Throughput increases linearly as additional nodes are added to the cluster. There is no predefined limit on the number of nodes that can be supported.

**Decentralized**
  All nodes are symmetric with no centralized service and no single point of failure.

**Partial Fault-Tolerance**
  A cluster management layer based on ZooKeeper re-routes events from a failed- to a other servers automatically. The state of the processing elements running on the failed server may be lost unless it is explicitly saved in a persistent store.

**Elastic**
  Computing load is automatically distributed on a heterogeneous cluster to maximize resource utilization and adapt to the skewness of the data without operator intervention even in the presence of hardware failures.

**Expandable**
  Applications can easily be written and deployed using a simple API. Many basic applications for stream processing are available and more are being written.

**Object Oriented**
  Internode communication using “plain old Java objects” (POJOs) with efficient serialization is fully supported. The application developer is not required to write schemas or use Hash Maps to send tuples across nodes. 

Design Principles
-----------------

* Adopt a functional programming style.
* Design a cluster with high availability that can scale using commodity hardware.
* Minimize latency by using local memory in each processing node and avoiding disk I/O bottlenecks.
* Use a decentralized and symmetric architecture; all nodes share the same functionality and responsibilities. There is no central node with specialized responsibilities. This greatly simplifies deployment and maintenance.
* Use a pluggable architecture to keep the design as generic and customizable as possible. 

Architecture Overview
---------------------

S4 is logically a message-passing system: computational units, called Processing Elements (PEs), send and receive messages (called Events). The S4 framework defines an API which every PE must implement, and provides facilities instantiating PEs and for transporting Events.

Events
^^^^^^

Events in S4 are arbitrary Java Objects that can be passed between PEs. Adapters convert external data sources into Events that S4 can process. Attributes of events can be accessed via getters in PEs. This is the exclusive mode of communication between PEs.

Events are dispatched in named streams. These streams are identified by a string-valued stream name

Processing Elements
^^^^^^^^^^^^^^^^^^^

Processing Elements (PEs) are the basic computational units in S4. They consume events and can in turn emit new events and update their state. Each instance of a PE is uniquely identified by four components:

* its **functionality** as defined by a PE class and associated configuration,
* the **named stream** that it consumes,
* the **keyed attribute** in those events, and
* the **value** of the keyed attribute in events which it consumes. 

Every PE consumes exactly those events which correspond to the value on which it is keyed. It may produce output events. Note that a PE is instantiated for each value of the key attribute. This instantiation is performed by the platform. For example, in an imaginary word counting application, ``WordCountPE`` is instantiated for each word in the input. When a new word is seen in an event, S4 creates a new instance of the PE corresponding to that word.

Several PEs are available for standard tasks such as count, aggregate, join, and so on. Many tasks can be accomplished using standard PEs which require no additional coding. The task is defined using a configuration file. Custom PEs can easily be programmed using the S4 software development tools.

Special Types of PEs
""""""""""""""""""""

**Keyless PE**
    A PE with no keyed attribute or value. These PEs consume all events on the stream with which they are associated. Keyless PEs are typically used at the input layer of an S4 cluster where events are yet to be assigned a key.

**PE Prototype**
    A PE with only the first three components of its identity (functionality, stream name, keyed attribute); the attribute value is unassigned. This object is configured upon initialization and, for any value V, it is capable of cloning itself to create fully qualified PEs of that class with identical configuration and value V for the keyed attribute. In S4, PEs are instantiated by this process. 

Processing Nodes
^^^^^^^^^^^^^^^^

Processing Nodes (PNs) are the logical hosts to PEs. They are responsible for listening to events, executing operations on the incoming events, dispatching events with the assistance of the communication layer, and emitting output events. S4 routes each event to PNs based on a hash function of the values of all known keyed attributes in that event. A single event may be routed to multiple PNs. The set of all possible keying attributes is known from the configuration of the S4 cluster. An event listener in the PN passes incoming events to the processing element container (PEC) which invokes the appropriate PEs in the appropriate order.

Every keyless PE that is configured in an application is instantiated once per PN. Similarly, one instance of each configured PE prototype exists in each PN.

References
----------

.. [1] **J. Dean and S. Ghemawat**, *MapReduce: simplified data processing on large clusters*, Commun. ACM, vol. 51, no. 1, pp. 107–113, 2008.
.. [2] **G. Agha**, *Actors: A Model of Concurrent Computation in Distributed Systems*, Cambridge, MA, USA: MIT Press, 1986.
