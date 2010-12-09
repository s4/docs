.. index:: application, example, speech01, SentenceReceiverPE

Getting Events into S4
======================

The simplest S4 application receives an event and prints it to standard output. Such an application can be found in `examples <https://github.com/s4/examples/tree/master/speech01>`_. The *speech01* application defines a single :term:`Processing Element` (PE) that listens to events and simply prints them as they arrive.

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

* ``AbstractPE``: Most PEs extend ``AbstractPE``. ``AbstractPE`` has some useful features, including:

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

Preparing to build the *speech01* example
-----------------------------------------

This section assumes you have set up your S4 runnable image as described in :doc:`/tutorials/getting_started`

To make the step-by-step instructions usable in your environment, set the following environmental variables as:

===============  =======================================================================================================================================================
variable name    value
===============  =======================================================================================================================================================
``SOURCE_BASE``  the base directory in which you clone the `examples repository <https://github.com/s4/examples>`_
``IMAGE_BASE``   the base directory of your runnable image (this would be :file:`${HOME}/s4image` if you followed the steps in :doc:`/tutorials/getting_started`)
===============  =======================================================================================================================================================

To compile the *speech01* example (or any S4 application), you need the ``s4_core`` jar file in your local maven repository. See :doc:`/manual/installing_core_locally`.

Building and running the *speech01* example
-------------------------------------------

To run the *speech01* example, do the following:

1. If you haven't yet cloned the examples repository, do the following:

  * ``cd ${SOURCE_BASE}``
  * ``git clone https://github.com/s4/examples.git``
2. ``cd ${SOURCE_BASE}/examples/speech01``
3. Build (follow instructions in `README <https://github.com/s4/examples/blob/master/speech01/README.md>`_)
4. ``cd ${IMAGE_BASE}/s4_apps``
5. ``tar xzf ${SOURCE_BASE}/examples/speech01/target/speech01-*.tar.gz``
6. ``cd ../bin``
7. start S4: ``s4_start.sh &``
8. Pipe the first ten lines of a sample input file into the load generator:

.. code-block:: bash

  head -10 ${SOURCE_BASE}/examples/testinput/speeches.txt | \
  generate_load.sh -x -r 2 -u ../s4_apps/speech01/lib/speech01-0.0.0.1.jar -
  
This command will emit events at roughly 2 events per second (as specified by ``-r 2``).

You should see messages like the following on standard output::

  Sentence is 'Four score and seven years ago our fathers brought forth on this continent a new nation, conceived in liberty and dedicated to the proposition that all men are created equal.', location null
  Sentence is 'Now we are engaged in a great civil war, testing whether that nation or any nation so conceived and so dedicated can long endure.', location null
  Sentence is 'We are met on a great battlefield of that war.', location null
  Sentence is 'We have come to dedicate a portion of that field as a final resting-place for those who here gave their lives that that nation might live.', location null
  Emitted 9 events

The load generator emits 9 events, 4 of which are ``Sentence`` events. Therefore, the S4 node should print four ``"Sentence is..."`` messages.

Note that each message ends with ``"location null"``. This is because the location field of each Sentence event is null. We will rectify that by joining two streams in :doc:`/manual/joining_streams`.
