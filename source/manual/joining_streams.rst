.. index:: application, example, speech02, PE prototype, keyed PE, key-less PE, join, TTL

Joining and Rerouting Streams
===============

Introduction
------------

This document describes how to use the ``JoinPE`` class to join two or more streams of events. We'll use the `speech02 example application <https://github.com/s4/examples/tree/master/speech02>`_ to illustrate its usage.

In describing the ``JoinPE`` class and how it is used in the *speech02* application, this document also describes several other basic concepts as well: PE prototypes, key-less PEs, keyed PEs, and TTLs.

In :doc:`/manual/getting_events_into_s4`, we showed how one introduces an event into an S4 node cluster. In the *speech01* example application, a single key-less PE instance prints each Sentence event it receives. From the printed output, you can see that the location field of the Sentence event is null.

In the `speech02 example application <https://github.com/s4/examples/tree/master/speech02>`_, each Sentence event is joined with its corresponding Speech event to obtain the location information.


A note on speeches.txt
----------------------

In :doc:`/manual/getting_events_into_s4`, you used :file:`speeches.txt` as input to the load generator. :file:`speeches.txt` contains a series of Speech, Sentence, and Highlight events that can be replayed using :file:`generate_load.sh`. The events have these attributes:

*Speech*

==============    =======     ====================================================
attribute name    type        notes
==============    =======     ====================================================
id                long        The id of the speech
location          String      The original location where the speech was delivered
speaker           String      The original deliverer of the speech
time              long        The start time of the recitation of the speech
==============    =======     ====================================================

*Sentence*

==============    =======     ====================================================
attribute name    type        notes
==============    =======     ====================================================
id                long        The id of the sentence
speechId          long        The id of the speech to which this sentence belongs
time              long        The start time of the recitation of the sentence
location          String      The original location where the associated speech was
                              delivered (not set in :file:`speeches.txt`)
==============    =======     ====================================================

*Highlight*

==============    =======     ====================================================
attribute name    type        notes
==============    =======     ====================================================
sentenceId        long        The id of the sentence being highlighted
time              long        The time at which the sentence was highlighted
==============    =======     ====================================================

The story of the events in :file:`speeches.txt` is as follows:

* Anonymous actors read famous speeches out loud. Each actor chooses a speech and recites it. These recitations are captured live and the audio is streamed to listeners.
* When the audio stream for a given speech starts, a ``Speech`` event is generated. The ``time`` field in the event indicates the time at which the audio for that speech began streaming, which is also roughly the time the event was generated.
* As a reciter completes each sentence in the speech, a ``Sentence`` event is generated. The ``time`` field in the event indicates the time at which the reciter finished uttering the sentence. The event also contains the id of the associated ``Speech`` event.
* Recitations can overlap. At any given time, it's possible that multiple speeches are being streamed to listeners.
* While the listener listens, he/she can also see a live transcription of the speech. When a listener finds a sentence interesting, he/she can click on the sentence in the live transcription. This generates a ``Highlight`` event. The ``time`` field in the event indicates the time at which the listener clicked the sentence. The event also contains the id of the sentence the listener found interesting.

Event Flow
----------

Overview
^^^^^^^^

In the :ref:`speech01 application <a_simple_pe>`, the flow is simple:

.. image:: /../_static/speech01_flow.png

The adapter -- in this case, :file:`generate_load.sh` -- emits key-less events on the RawSentence stream. Because eventCatcher PE registers its interest in the RawSentence stream, regardless of key, it receives those events.

As mentioned in :doc:`/manual/getting_events_into_s4`, the adapter will evenly distribute events amongst the nodes of the S4 cluster. Because the eventCatcherPE is key-less, there is at most one instance of the PE per S4 node. If your S4 cluster contains only one node, then all events in the RawSentence stream will go to the single eventCatcher PE instance.

The flow for speech02 is more complex:

.. image:: /../_static/speech02_flow.png

reroute PEs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Like the eventCatcher PE in *speech01*, rerouteSentencePE and rerouteSpeechPE serve as entry points to the S4 cluster. They both listen to key-less events from the outside world, in this case the adapter. 

Here's the configuration for the rerouteSentencePE:

.. code-block:: xml

  <bean id="rerouteSentencePE" class="io.s4.processor.ReroutePE">
    <property name="id" value="rerouteSentencePE"/>
    <property name="dispatcher" ref="dispatcher"/>
    <property name="keys">
      <list>
        <value>RawSentence *</value>
      </list>
    </property>
    <property name="outputStreamName" value="Sentence"/>
  </bean>

The ``io.s4.processor.ReroutePE`` class is provided by the platform. This class simply receives events and re-emits events them on a specified stream. This functionality is typically used to convert a key-less stream from an adapter into a keyed stream.

In this case, the rerouteSentencePE listens to events on the RawSentence stream and re-emits them on the Sentence stream. The dispatcher is configured to dispatch such events using the speech id as the dispatch key. So the output from the rerouteSentencePE will be keyed events. How the events obtain keys is described in :doc:`dispatcher`.

rerouteSpeechPE is similar, except it listens to events on the RawSpeech stream and re-emits them on the Speech stream:

.. code-block:: xml

 <bean id="rerouteSpeechPE" class="io.s4.processor.ReroutePE">
  <property name="id" value="rerouteSpeechPE"/>
  <property name="dispatcher" ref="dispatcher"/>
  <property name="keys">
    <list>
      <value>RawSpeech *</value>
    </list>
  </property>
  <property name="outputStreamName" value="Speech"/>
 </bean>


Join PE
^^^^^^^

sentenceJoinPE listens to the output of the reroute PEs. The reroute PEs will emit ``Speech`` events on the Speech stream, and ``Sentence`` events on the Sentence stream.

Here's the configuration for the Join PE:

.. code-block:: xml

	<bean id="sentenceJoinPE" class="io.s4.processor.JoinPE">
	  <property name="id" value="sentenceJoinPE"/>
	  <property name="keys">
	    <list>
	      <value>Sentence speechId</value>
	      <value>Speech id</value>
	    </list>
	  </property>
	  <property name="includeFields">
	    <list>
	      <value>Sentence *</value>
	      <value>Speech location</value>
	    </list>
	  </property>
	  <property name="outputStreamName" value="SentenceJoined"/>
	  <property name="outputClassName" value="io.s4.example.speech01.Sentence"/>
	  <property name="dispatcher" ref="dispatcher"/>
	  <property name="ttl" value="600"/> <!-- join related events that arrive no more than 10 minutes apart -->
	</bean>

Here we define a :term:`PE prototype`. As mentioned in :doc:`/manual/overview`, a PE prototype is identified within S4 by three components: Functionality, stream name(s), and key attribute. In the case of sentenceJoinPE, here are the three components:

=====================   =====================================================
identity component      value
=====================   =====================================================
Functionality           * Class: ``io.s4.processor.JoinPE``
                        * Configuration:
                           * includeFields=[Sentence \*","Speech location"]
                           * outputStreamName="SentenceJoined"
                           * ttl=600
                           * etc.
Stream name(s)          * Sentence
                        * Speech
Key attribute           speech id (field ``speechId`` in the Sentence stream, field ``id`` in the Speech stream)
=====================   =====================================================

Note that the key attribute is specified by two separate fields: ``speechId`` in the Sentence stream, and ``id`` in the Speech stream. Think of speechId as a "foreign key" referring to the ``id`` field of some event in the Speech stream. That is, both fields contain the id of some Speech.

S4 will create one ``JoinPE`` instance for each value of speech id encountered. Therefore, each PE instance will have four identity components: The same three identity components of its prototype, plus the value of the key attribute (i.e., the value of speech id). When S4 receives an event with speech id *n* from either the Sentence or Speech streams, the following happens:
 
* If an instance for key value *n* does not already exist, S4 creates one by cloning the prototype
* The event is passed to the PE instance for key value *n*.

Say S4 encounters the speech ids 12000000, 22000000, 24000000, and 30000000 in the Sentence and Speech streams. Then following PE instances would exist:

.. image:: /../_static/joinPEa.png

In the likely scenario where there are multiple S4 nodes, it may look like this:

.. image:: /../_static/joinPEb.png

Because PE instances are indexed by a key attribute, the Speech event for key value *n* and the Sentence events for key value *n* will all go to the same PE instance: the instance keyed by value *n*. Since related Speech and Sentence events are arriving to the same PE instance, the PE can join the related events.

``JoinPE`` sets aside one slot for each incoming stream specified in the ``includeFields`` property. When an event on stream *s1* arrives to the PE instance, the event is put in the slot for stream *s1*. When an event on stream *s2* arrives to the PE instance, the event is put in the slot for stream *s2*, and so on. When all slots contain an event, ``JoinPE`` creates a new event and emits it. ``JoinPE`` creates the new event as follows:

1. Create an instance of the class specified by ``outputClassName``.
2. For each slot, copy the specified fields from the contained event to the new event
3. Emit the new event on the stream specified by ``outputStreamName``.

If all slots are already full and a new event comes along, the corresponding slot is updated and a new event emitted. Therefore, a single ``JoinPE`` instance can emit multiple events. That is, it handles one-one, one-many, and many-many joins.

In the case of sentenceJoinPE, there are only two slots: one for the Sentence stream and one for the Speech stream. It's also a one-many join: That is, there will be many ``Sentence`` events associated with each ``Speech`` event. sentenceJoinPE basically implements this logic:

.. code-block:: sql

   select Sentence.*, Speech.location
   into SentenceJoined
   from Sentence, Speech
   where Sentence.speechId = Speech.id

sentenceJoinPE creates new ``Sentence`` events which are the same as the incoming ``Sentence`` events, except with the location field filled in.

Here's a typical flow for the sentenceJoinPE:

#. A ``Speech`` event for speech id 11 arrives on the Speech stream.
#. Because no sentenceJoinPE exists for speech id 11, S4 creates one by cloning the prototype.
#. S4 calls the instance's processEvent() method.
#. The PE instance stores the event in the slot for stream Speech.
#. 10 seconds later, a ``Sentence`` event for speech id 11 arrives on the Sentence stream.
#. S4 locates the sentenceJoinPE instance for speech id 11.
#. S4 calls the instance's processEvent() method.
#. The PE instance stores the event in the slot for stream ``Sentence``. Because all slots are full, the PE instance does the following:

   #. Creates a new ``Sentence`` object.
   #. Copies all fields from the old ``Sentence`` event into the new ``Sentence`` event.
   #. Copies the ``location`` field from the ``Speech`` event into the new ``Sentence`` event.
   #. Emits the new ``Sentence`` event onto the SentenceJoined stream.
9. Four seconds later,  another ``Sentence`` event for speech id 11 arrives on the Sentence stream.
#. S4 locates the sentenceJoinPE instance for speech id 11.
#. S4 calls the instance's processEvent() method.
#. The PE instance replaces the existing event in the slot for stream Sentence with the newly arrived event. Because all slots are full, the PE instance repeats the above steps for emitting a new event.


sentenceJoinPE's ``ttl`` property is set to 600 seconds (10 minutes). The framework will consider the PE instance for speech id *n* dead if that instance receives no events for 10 minutes. If an event for speech id *n* arrives after that 10-minute period of idleness, then a new instance for value *n* will be created with all slots reset. Therefore, a join succeeds only if the related events arrive within 10 minutes of each other.

The sentenceJoinPE uses the configured dispatcher to dispatch the events to the appropriate nodes. The dispatcher is described in :doc:`dispatcher`.

Join PE Caveats
^^^^^^^^^^^^^^^

The ``JoinPE`` will fail to join properly if multiple events arrive to one slot and some of the other slots are empty.

Using the speech02 application as an example, consider this case:

#. A ``Sentence`` event for speech id 11 arrives on the Speech stream.
#. Because no sentenceJoinPE exists for speech id 11, S4 creates one by cloning the prototype.
#. S4 calls the instance's processEvent() method.
#. The PE instance stores the event in the slot for stream Sentence.
#. 10 seconds later, another ``Sentence`` event for speech id 11 arrives on the Sentence stream.
#. S4 locates the sentenceJoinPE instance for speech id 11.
#. S4 calls the instance's processEvent() method.
#. The PE instance replaces the existing event in the slot for stream Sentence with the newly arrived event. The old event is forgotten without ever being joined to its corresponding Speech event.
#. A few seconds later, a ``Speech`` event for speech id 11 arrives on the Speech stream.
#. S4 locates the sentenceJoinPE instance for speech id 11.
#. S4 calls the instance's processEvent() method.
#. The PE instance stores the event in the slot for stream Speech.  Because the all slots are full, the PE instance emits a new event.

In this case, two Sentence events arrived before the Speech event arrived. As a result, one of the Sentence events was not joined to its corresponding Speech event.

If you play back the :file:`speeches.txt` file at a high enough rate and use multiple S4 nodes, you will see cases of this, even though no such ordering can be found in the file.

reroute PEs revisited
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As with the sentenceJoinPE definition, the rerouteSentencePE definition creates an instance of a :term:`PE Prototype`. In this case, however, it has only 2 identity components:

=====================   =====================================================
identity component      value
=====================   =====================================================
Functionality           * Class: ``io.s4.processor.ReroutePE``
                        * Configuration:
                           * outputStreamName="Sentence"
                           * etc.
Stream name(s)          * RawSpeech
Key attribute           None, this is a prototype for a key-less PE
=====================   =====================================================

Because the PE is key-less, there is at most one instance per S4 node:

.. image:: /../_static/joinPEc.png

.. _building_and_running_speech02:

Building and running the *speech02* example
-------------------------------------------

This section assumes you have first built the *speech01* example application according to :doc:`/manual/getting_events_into_s4`.

To run the *speech02* example, do the following:

1. Kill any previous instance of S4 you might have running
2. Remove any extraneous applications: ``rm -fr $S4_IMAGE/s4-apps/*``
3. Clean out your logs directory: ``rm $S4_IMAGE/s4-core/logs/s4-core/*``
4. Copy the application into :file:`s4-apps`: ``cp -r $S4_IMAGE/s4-example-apps/s4-example-speech02 $S4_IMAGE/s4-apps/`` 
5. start S4: ``$S4_IMAGE/scripts/start-s4.sh -r client-adapter &``
6. start the adapter:

.. code-block:: bash

   $S4_IMAGE/scripts/run-client-adapter.sh -s client-adapter \
   -g s4 -x -d $S4_IMAGE/s4-core/conf/default/client-stub-conf.xml &

7. Pipe the first ten lines of a sample input file into the load generator:

.. code-block:: bash

  head -10 $S4_IMAGE/s4-example-testinput/speeches.txt | \
  sh $S4_IMAGE/s4-tools-loadgenerator/scripts/generate-load.sh -r 2 -
  
This command will emit events at roughly 2 events per second (as specified by ``-r 2``).

You should see messages like the following on standard output::

   Sentence is 'Four score and seven years ago our fathers brought forth on this continent a new nation, conceived in liberty and dedicated to the proposition that all men are created equal.', location gettysburg, pa, us
   Sentence is 'Now we are engaged in a great civil war, testing whether that nation or any nation so conceived and so dedicated can long endure.', location gettysburg, pa, us
   Sentence is 'We are met on a great battlefield of that war.', location gettysburg, pa, us
   Sentence is 'We have come to dedicate a portion of that field as a final resting-place for those who here gave their lives that that nation might live.', location gettysburg, pa, us
   Emitted 9 events

*speech02* produces the same messages as the *speech01* example application, except the location field is now filled in.
