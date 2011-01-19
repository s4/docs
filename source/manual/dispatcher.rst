.. index:: dispatcher, keys, emit

Dispatcher
==========

Overview
--------

PE instances use a dispatcher to get events to other PE instances; That is, PEs use a dispatcher to emit events. A dispatcher delivers events to PE instances, even if those instances reside on other S4 nodes in the cluster. 

Using the dispatcher
--------------------

One example of a PE that uses a dispatcher is ``io.s4.processor.ReroutePE``, a platform-provided class. It uses the dispatcher as `follows <https://github.com/s4/core/blob/master/src/main/java/io/s4/processor/ReroutePE.java#L105>`_:

.. code-block:: java

   dispatcher.dispatchEvent(outputStreamName, newEvent);

where ``dispatcher`` is a reference to some configured dispatcher.

The caller of  Dispatcher#dispatchEvent passes the event and the stream name, but does not specify the keys on which the event will be dispatched. That decision is made by the dispatcher based on its configuration.

You configure the dispatcher with the keys on which it will dispatch events. You can specify a different set of keys for each stream.

For example, in the `speech002 <https://github.com/s4/examples/tree/master/speech02>`_ application, the dispatcher is configured to dispatch events as follows:

========================  =============================
Stream name               key on which to dispatch event
========================  =============================
Sentence, SentenceJoined  speechId
Speech                    id
========================  =============================

That is, if the the event is passed to dispatcher#dispatchEvent with a stream name of ``Sentence`` or ``SentenceJoined``, the dispatcher will dispatch the event based on the content of the event's ``speechId`` field. If the event does not have a ``speechId`` field, or the field's value is null, the event is ignored.

The dispatcher will annotate the event with information about the key that was used for dispatching. Then the dispatcher will send the event to the S4 node base on the value of the key. The receiving S4 node uses the annotation to determine which local PE instances should receive the event.

Here's how the dispatcher configuration looks in the `speech002 <https://github.com/s4/examples/tree/master/speech02>`_ application:

.. code-block:: xml

	<bean id="dispatcher" class="io.s4.dispatcher.Dispatcher" init-method="init">
	  <property name="partitioners">
	    <list>
	      <ref bean="sentenceSpeechIdPartitioner"/>
	      <ref bean="speechIdPartitioner"/>
	    </list>
	  </property>
	  <property name="eventEmitter" ref="commLayerEmitter"/>
	  <property name="loggerName" value="s4"/>
	</bean>

The configuration for the dispatcher itself is quite short: It specifies a list of partitioner objects (each of which you also configure), and a reference to the commLayerEmitter component. You do not need to create the commLayerEmitter component: it is instantiated by S4 runtime on startup.

Here's the configuration for the two partitioners:

.. code-block:: xml

	<bean id="sentenceSpeechIdPartitioner" class="io.s4.dispatcher.partitioner.DefaultPartitioner">
	  <property name="streamNames">
	    <list>
	      <value>Sentence</value>
	      <value>SentenceJoined</value>
	    </list>
	  </property>
	  <property name="hashKey">
	    <list>
	      <value>speechId</value>
	    </list>
	  </property>
	  <property name="hasher" ref="hasher"/>
	  <property name="debug" value="false"/>
	</bean>

	<bean id="speechIdPartitioner" class="io.s4.dispatcher.partitioner.DefaultPartitioner">
	  <property name="streamNames">
	    <list>
	      <value>Speech</value>
	    </list>
	  </property>
	  <property name="hashKey">
	    <list>
	      <value>id</value>
	    </list>
	  </property>
	  <property name="hasher" ref="hasher"/>
	  <property name="debug" value="false"/>
	</bean>

The partitioners specify the keys on which the associated dispatcher will dispatch events. Each partitioner is an instance of ``DefaultPartitioner``, a class that is provided by the platform. Besides specifying the stream names and keys, you also specify a reference to a hasher. The S4 runtime provides a hasher instance with the bean name ``hasher``. This hasher uses the FNV1-64 algorithm. However, you can swap it out for you own instance (as long as it implements the ``Hasher`` interface).

Compound keys, nested keys, and list keys are discussed in :doc:`specifying_keys`.





 
