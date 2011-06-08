.. index:: application, example, speech01, SentenceReceiverPE

Getting Events into S4
======================

The simplest S4 application receives an event and prints it to standard output. Such an application can be found in `examples <https://github.com/s4/examples/tree/master/speech01>`_. The *speech01* application defines a single :term:`Processing Element` (PE) that listens to events and simply prints them as they arrive.

.. _a_simple_pe:

A Simple PE
------------

Here's the meat of that PE:

.. code-block:: java

  public class SentenceReceiverPE extends AbstractPE {

    public void processEvent(Sentence sentence) {
        System.out.printf("Sentence is '%s', location %s\n", sentence.getText(),
                          sentence.getLocation());
    }

    @Override
    public void output() {
        // not called in this example
    }

    @Override
    public String getId() {
        return this.getClass().getName();
    }
  }

Let's quickly describe a few things about this PE class:

* ``AbstractPE``: PEs must extend ``AbstractPE``. ``AbstractPE`` has some useful features, including:

  * a method for determining the value on which the PE instance is keyed
  * a method for finding the object within the event that contains the key (useful if the key resides in a List)
  * a method for getting timing information
  * the ability to automatically invoke the output method based on time or number of events (see output below).

  The key-related features will become clear when we discuss how PEs are keyed.

* ``processEvent()``: The application programmer writes one ``processEvent()`` method for each type of event he wants the PE to handle. In this case, the developer has written only one ``processEvent()`` method to handle events of type Sentence.

  A catch-all ``processEvent()`` would take ``Object`` as a parameter.

  If the developer provides no ``processEvent()`` method, the PE may get instantiated, but it will never get invoked.

* ``output()``: You can configure the PE such that this method is called based on one of the following criteria:

  * ``output()`` is called every ``n`` events
  * ``output()`` is called on every ``n`` second boundary

Here's the Spring configuration for that PE, found in :file:`src/main/resources/speech01_conf.xml`:

.. code-block:: xml

  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.springframework.org/schema/beans             
         http://www.springframework.org/schema/beans/spring-beans-2.0.xsd">

    <bean id="eventCatcher" class="io.s4.example.speech01.SentenceReceiverPE">
      <property name="keys">
        <list>
          <value>RawSentence *</value>
        </list>
      </property>
    </bean>
  </beans>

This defines a PE *prototype* called ``eventCatcher`` of type ``SentenceReceiverPE``. A PE prototype is a special instance of the PE that is never invoked by the framework. Instead, it serves as an example instance from which actual PE instances are created (via ``clone()``). This means that PE instances are configured like their prototype: If a prototype has a reference to some object, so does its associated PE instances.

This PE is configured to listen to the ``RawSentence`` stream. The asterisk indicates that this PE does not care about the key on which the events are dispatched. This PE is thus *key-less*, and at most only one instance (besides the prototype) will get created on each S4 node. Since the PE is not keyed, each instance will receive all events on stream ``RawSentence`` that arrive at the corresponding S4 node.

Key-less PEs usually serve as entry point PEs: That is, the PEs that receive events from the outside world. This is because an adapter typically does not choose an S4 node based on a key in the event, but randomly (or in some other fashion that evenly distributes the events amongst the S4 nodes).

More about keys in the discussion about :doc:`/manual/joining_streams`.

Building and running the *speech01* example
-------------------------------------------

To run the *speech01* example, do the following:

1. Set up S4 according to :ref:`Set Up S4 <getting_started_set_up>`
2. Remove any extraneous applications: ``rm -fr $S4_IMAGE/s4-apps/*``
3. Copy the application into :file:`s4-apps`: ``cp -r $S4_IMAGE/s4-example-apps/s4-example-speech01 $S4_IMAGE/s4-apps/``
4. start S4: ``$S4_IMAGE/scripts/start-s4.sh &``
5. start the adapter:

.. code-block:: bash

   $S4_IMAGE/scripts/run-client-adapter.sh -s client-adapter \
   -g s4 -x -d $S4_IMAGE/s4-core/conf/default/client-stub-conf.xml &

6. Pipe the first ten lines of a sample input file into the load generator:

.. code-block:: bash

  head -10 $S4_IMAGE/s4-example-testinput/speeches.txt | \
  sh $S4_IMAGE/s4-tools-loadgenerator/scripts/generate-load.sh -r 2 -
  
This command will emit events at roughly 2 events per second (as specified by ``-r 2``).

You should see messages like the following on standard output::

  Sentence is 'Four score and seven years ago our fathers brought forth on this continent a new nation, conceived in liberty and dedicated to the proposition that all men are created equal.', location null
  Sentence is 'Now we are engaged in a great civil war, testing whether that nation or any nation so conceived and so dedicated can long endure.', location null
  Sentence is 'We are met on a great battlefield of that war.', location null
  Sentence is 'We have come to dedicate a portion of that field as a final resting-place for those who here gave their lives that that nation might live.', location null
  Emitted 9 events

The load generator emits 9 events, 4 of which are ``Sentence`` events. Therefore, the S4 node should print four ``"Sentence is..."`` messages.

Note that each message ends with ``"location null"``. This is because the location field of each Sentence event is null. We will rectify that by joining two streams in :doc:`/manual/joining_streams`.
