.. index:: glossary

Glossary
========

.. glossary::

  Processing Node
    Processing Nodes (PNs) are the logical hosts to PEs. They are responsible for listening to events, executing operations on the incoming events, dispatching events with the assistance of the communication layer, and emitting output events.

  Processing Element
    Processing Elements (PEs) are the basic computational units in S4. They consume events and can in turn emit new events and update their state. Each instance of a PE is uniquely identified by four components:

	* its **functionality** as defined by a PE class and associated configuration,
	* the **named stream** that it consumes,
	* the **keyed attribute** in those events, and
	* the **value** of the keyed attribute in events which it consumes.

  PE Prototype
    A PE with only the first three components of its identity (functionality, stream name, keyed attribute); the attribute value is unassigned. This object is configured upon initialization and, for any value V, it is capable of cloning itself to create fully qualified PEs of that class with identical configuration and value V for the keyed attribute. In S4, PEs are instantiated by this process. 

  PE Container
    The PE Container holds all PE instances, including the PE prototypes. The PE container is responsible for routing incoming events to the appropriate PE instances.
