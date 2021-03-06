Getting Started
===============

By default, S4 runs a cluster consisting of a single node. Also by default, S4 runs without a cluster management system (e.g., ZooKeeper). Overriding the defaults is fairly straightforward. 

The steps in this guide have been tested on:

* Red Hat Enterprise Linux 4
* Mac OS X Leopard

.. _getting_started_set_up:

Set Up S4
---------

* Clone ``s4`` to a source directory (``<source_base>``)

  * ``cd <source_base>``
  * ``git clone git://github.com/s4/s4.git``
* ``cd s4``
* ``git checkout tags/v0.3.0``
* ``./gradlew allImage``
* ``cd build/s4-image/``
* ``export S4_IMAGE=`pwd```

.. _getting_started_set_up_application:

Set up the Example Application
------------------------------

* Clone examples to a source directory (``<source_base>``)

  * ``cd <source_base>``
  * ``git clone git://github.com/s4/twittertopiccount.git``
* ``cd twittertopiccount``
* ``git checkout tags/v0.3.0``
* ``./gradlew install``
* Deploy the application: ``./gradlew deploy``
* ``cd build/install/twitter_feed_listener``
* ``export TWIT_LISTENER=`pwd```


Run
---

You should see a sample application under ``$S4_IMAGE/s4-apps`` called ``twittertopiccount``. This sample application listens to the Twitter Spritzer and keeps track of the top 10 hash tags. You can try it as follows:

* Set the environment variable ``JAVA_HOME``. S4 scripts expect to find the java command at ``${JAVA_HOME}/bin``.
* Start an S4 node:

  ``$S4_IMAGE/scripts/start-s4.sh -r client-adapter &``

This will print out some initial set up messages. The setup messages should end with something like this::

  [/home/robloc/githubimage/s4_apps/twittertopiccount/twittertopiccount_conf.xml]
  Adding processing element with bean name topicExtractorPE, id topicSeenPE
  adding pe: io.s4.example.twittertopiccount.TopicExtractorPE@6b496d
  Using ConMapPersister ..
  Adding processing element with bean name topicCountAndReportPE, id topicCountAndReportPE
  adding pe: io.s4.example.twittertopiccount.TopicCountAndReportPE@11f23e5
  Using ConMapPersister ..
  Adding processing element with bean name top10TopicPE, id top10TopicPE
  adding pe: io.s4.example.twittertopiccount.TopNTopicPE@2d189c
  Using ConMapPersister ..

Look in ``$S4_IMAGE/s4-core/logs/s4-core`` for the S4 node's log4j log.

* Run the adapter:

The adapter will adapt Twitter status messages into events expected by the sample application. To run it, enter this command:

.. code-block:: bash

   $S4_IMAGE/scripts/run-client-adapter.sh -s client-adapter \
   -g s4 -d $S4_IMAGE/s4-core/conf/default/client-stub-conf.xml &


* Start the Twitter feed listener. Replace <your-twitter-user> and <your-twitter-password> with a valid Twitter account userid and password:

.. code-block:: bash

   $TWIT_LISTENER/bin/twitter_feed_listener <your-twitter-user> <your-twitter-password> &

To check the current top 10 hash tags, look in this file: :file:`/tmp/top_n_hashtags`. Note, twittertopiccount will not include any hash tag that has less that 4 references. Therefore, it might take a few minutes for this file to appear.

Troubleshooting
---------------

* Process hanging waiting for lock

When running in red button mode (i.e., not using Zookeeper as a cluster manager), s4 processes use lock files in the ``$S4_IMAGEs4_core/lock`` directory. If you've killed any s4 Java process with the ``KILL`` (9) signal, the lock file for that Java process may not get cleared out. Therefore, subsequent load generator or s4 node processes may hang waiting for the lock. You will see a message like the following::

   Process taken up by another process lockFile:/Users/robbins/github/s4/build/s4-image/scripts/../s4-core/lock/s4s4node-0
   processAvailable:false

To avoid this issue, make sure you always use kill with the default signal. If you are running the process in the foreground, :kbd:`Control-c` also works fine.

If you run into trouble with lock files:
   
  * Kill all s4 processes (including the adapter)
  * Clear all files in ``$S4_IMAGE/s4-core/lock``
  * Try running the processes again

