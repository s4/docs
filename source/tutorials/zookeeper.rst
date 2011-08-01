.. index:: zookeeper

Running S4 with Zookeeper
=========================

This document describes how to run an S4 cluster in dynamic mode. In dynamic mode, S4's communication layer uses Zookeeper to manage the topology of the S4 cluster.

This document assumes that you set up S4 according to :ref:`Set Up S4 <getting_started_set_up>`.

Setting up your Zookeeper instance
----------------------------------

1. Install `Zookeeper <http://zookeeper.apache.org/>`_.
2. Start up your Zookeeper instance
3. Load the clusters.xml file from the dynamic configuration into zookeeper:

.. code-block:: bash

   $S4_IMAGE/scripts/task-setup.sh localhost:2181 clean setup $S4_IMAGE/s4-core/conf/dynamic/clusters.xml

Change the first parameter if your Zookeeper instance is not on the host from which the command is run, or if you've changed Zookeeper's default port.

Note that when this command loads the file into Zookeeper, the <machine> elements within that file are ignored.

Run twittertopiccount example in dynamic mode
---------------------------------------------

1. Run an S4 instance using the dynamic configuration:

.. code-block:: bash

   $S4_IMAGE/scripts/start-s4.sh dynamic &

2. To check that the S4 node grabbed a task in Zookeeper, run the Zookeeper client utility (zkCli.sh) and issue the following command:

.. code-block:: text

   ls /s4/s4/process

You should see one znode called task-0. If you kill the S4 node process, you should see this znode go away.

3. Run an client adapter using the dynamic configuration:

.. code-block:: bash

   $S4_IMAGE/scripts/run-client-adapter.sh -s client-adapter -g s4 -d $S4_IMAGE/s4-core/conf/dynamic/client-stub-conf.xml dynamic &

4. To check that the client adapter node grabbed a task in Zookeeper, run the Zookeeper client utility (zkCli.sh) and issue the following command:

.. code-block:: text

   ls /client-adapter/s4/process

5. Start the twitter feed reader which will send events to the client-adapter process (remember to fill in your twitter userid and password on the command line):

.. code-block:: bash

   $TWIT_LISTENER/bin/twitter_feed_listener <your-twitter-user> <your-twitter-password> &

Check that :file:`/tmp/top_n_hashtags` is getting updated

Here there are two clusters: the client adapter cluster (named, appropriately, "client-adpater") and the s4 cluster (named "s4"). The client adapter cluster receives events from a client process (in this case, the twitter_feed_listener program) and forwards them to the s4 cluster.

In this case the two clusters have one node each. The two clusters can talk back and forth, but in the twittertopiccount example the talk is one way: the client adapter sends events to the s4 cluster.

You can update the dynamic configuration (or create your own configuration) to include multiple s4 and/or client adapter nodes. You need to start as many s4 node processes as there are s4 node entries in the configuration (see :doc:`/tutorials/multiple_nodes`). If you start more nodes than there are entries in the configuration, the additional processes become standby processes: If a running node fails, one of the standby processes will take over that partition id.

If you are running the nodes across multiple machines, you need to change zk_address in the configuration's :file:`s4-core.properties-header` to the specific machine on which Zookeeper is running (rather than localhost). If you have a Zookeeper cluster, specify all the machine:port combinations in a comma separated list.




