.. index:: multiple nodes

Running S4 with Multiple Nodes
==============================

This document describes how to run an S4 cluster with more than one node. This document assumes you are running S4 in red button mode (i.e., running without a cluster manager like Zookeeper).

This document also assumes that the IMAGE_BASE environment variable contains the location of your s4 runnable image. If you set up the runnable image according to :doc:`/tutorials/getting_started`, you would set the variable to ``${HOME}/s4image``.

Quick cluster configuration overview
------------------------------------
A cluster configuration file describes the nodes in an S4 node cluster, an adapter cluster, or both. The communication layer uses this configuration to assign tasks and find nodes.

The default configuration can be found at ``${IMAGE_BASE}/s4_core/conf/redbutton/clusters.xml``:

.. code-block:: xml
  :linenos:

  <config version="-1">
    <cluster name="s4" type="s4" mode="unicast">
      <node>
        <partition>0</partition>
        <machine>localhost</machine> <!-- used only in red button mode -->
        <port>5077</port>
        <taskId>s4node-0</taskId>
      </node>
    </cluster>
    <cluster name="s4" type="adapter" mode="unicast">
      <node>
        <machine>localhost</machine> <!-- used only in red button mode -->
        <taskId>adapter-0</taskId>
      </node>
    </cluster>
  </config>

This configuration file describes two clusters: one s4 node cluster and one adapter cluster. Both clusters contain only a single node. Both clusters use unicast for inter-node communication. Both clusters are named ``s4``: It's OK for two clusters to have the same name as long as they are of different types.

Under each cluster element are node elements. Each node element describes a node in the cluster. The node element has the following child elements:

* **partition**

  The partition that the node represents.

  All partitions 0-*n* should be represented in the configuration, where *n* is the ((number of nodes)-1).

  This element is valid for S4 nodes only.

* **machine**
  
  The machine on which the node will run.

  In red button node, this setting is respected. However, it is ignored when a cluster management system like Zookeeper is used. In that case, each S4 node grabs a task at start-up.

* **port**

  The port on which the S4 node listens. This should not be specified for adapter nodes.

* **taskId**

  This is a string that names the task taken by the S4 or adapter node. For an S4 node, it is usually s4node-*partition*, but any string will do. The names should be unique within a cluster.

  In red button mode, this string is used to form a lock filename. Therefore, you should use only characters that are valid for your filesystem.

Adding another S4 node
----------------------

Lets say you want to run two S4 nodes on your machine. First, create a new configuration directory:

* ``cd ${IMAGE_BASE}/s4_core/conf``
* ``cp -r redbutton myconfig``

Edit ``myconfig/clusters.xml`` and add a new S4 node: 

.. code-block:: xml
  :linenos:

  <config version="-1">
    <cluster name="s4" type="s4" mode="unicast">
      <node>
        <partition>0</partition>
        <machine>localhost</machine> <!-- used only in red button mode -->
        <port>5077</port>
        <taskId>s4node-0</taskId>
      </node>
      <node> <!-- this is the new node element -->
        <partition>1</partition>
        <machine>localhost</machine> <!-- used only in red button mode -->
        <port>5078</port>
        <taskId>s4node-1</taskId>
      </node>
    </cluster>
    <cluster name="s4" type="adapter" mode="unicast">
      <node>
        <machine>localhost</machine> <!-- used only in red button mode -->
        <taskId>adapter-0</taskId>
      </node>
    </cluster>
  </config>

Since both nodes will run on the same machine (``localhost``), make sure the two nodes listen on different ports.

Now run the sample application, this time using your new configuration:

* ``cd ${IMAGE_BASE}/bin``
* clean out your logs directory (to better demonstrate the example): ``rm -fr ../s4_core/logs/*``
* Start the first S4 node and tell it to use your configuration: ``s4_start.sh myconfig &``
* Start the second S4 node, also using your configuration: ``s4_start.sh myconfig &``

  * Note: When running multiple nodes in red button mode on a single machine, always start them from the same ``${IMAGE_BASE}/bin``
  * Also note: If you start a third S4 node, its communication layer will not find an available task. Therefore, it will just wait.
* Start the adapter and tell it to use your configuration:

.. code-block:: bash

  run_adapter.sh -u ../s4_apps/twittertopiccount/lib/twittertopiccount-*.jar \
  -d ../s4_apps/twittertopiccount/adapter_conf.xml myconfig &

If you get an exception, make sure you have the latest version of ``run_adapter.sh``.

* Check that events are getting evenly distributed amongst the two nodes:

.. code-block:: bash

  find ../s4_core/logs/s4_core -name "s4_core_*.log" -print -exec sh -c 'grep -i "count by" {} | tail -4' \; 

You should see something like the following::

  ../s4_core/logs/s4_core/s4_core_29230.log
  2010-11-19 22:57:37,240 s4 INFO (PEContainer.java:285) Count by RawStatus : 588
  2010-11-19 22:57:37,240 s4 INFO (PEContainer.java:285) Count by TopicSeen topic: 117
  2010-11-19 22:57:47,243 s4 INFO (PEContainer.java:285) Count by RawStatus : 632
  2010-11-19 22:57:47,243 s4 INFO (PEContainer.java:285) Count by TopicSeen topic: 125
  2010-11-19 22:57:57,245 s4 INFO (PEContainer.java:285) Count by RawStatus : 688
  2010-11-19 22:57:57,246 s4 INFO (PEContainer.java:285) Count by TopicSeen topic: 135
  ../s4_core/logs/s4_core/s4_core_29131.log
  2010-11-19 22:57:43,368 s4 INFO (PEContainer.java:285) Count by RawStatus : 611
  2010-11-19 22:57:43,368 s4 INFO (PEContainer.java:285) Count by TopicSeen topic: 96
  2010-11-19 22:57:43,368 s4 INFO (PEContainer.java:285) Count by AggregatedTopicSeen reportKey: 45
  2010-11-19 22:57:53,372 s4 INFO (PEContainer.java:285) Count by RawStatus : 662
  2010-11-19 22:57:53,372 s4 INFO (PEContainer.java:285) Count by TopicSeen topic: 104
  2010-11-19 22:57:53,373 s4 INFO (PEContainer.java:285) Count by AggregatedTopicSeen reportKey: 51

One node has received 688 events from the adapter on the ``RawStatus`` stream, and the other node has received 662. For events originating from the S4 nodes themselves (the ``TopicSeen`` stream), one node has received 135 events, and the other has received 104. So the events are getting fairly evenly distributed in this example. Note that only one node is getting events from the ``AggregatedTopicSeen`` stream: that is expected in the ``twittertopiccount`` application.

Running S4 nodes on multiple machines
-------------------------------------

To spread the nodes across multiple machines, specify the machine names in the ``<machine>`` elements of ``myconfig/clusters.xml``, e.g.

.. code-block:: xml
  :linenos:
  
  <config version="-1">
    <cluster name="s4" type="s4" mode="unicast">
      <node>
        <partition>0</partition>
        <machine>machine1.s4.io</machine> <!-- used only in red button mode -->
        <port>5077</port>
        <taskId>s4node-0</taskId>
      </node>
      <node> <!-- this is the new node element -->
        <partition>1</partition>
        <machine>machine2.s4.io</machine> <!-- used only in red button mode -->
        <port>5078</port>
        <taskId>s4node-1</taskId>
      </node>
    </cluster>
    <cluster name="s4" type="adapter" mode="unicast">
      <node>
        <machine>machine3.s4.io</machine> <!-- used only in red button mode -->
        <taskId>adapter-0</taskId>
      </node>
    </cluster>
  </config>

In this example, the S4 node for partition 0 will run on ``machine1.s4.io``. The S4 node for partition 1 will run on ``machine2.s4.io``. The adapter will run on ``machine3.s4.io``.

Let's run the nodes on three machines:

* Choose three machines. I will call them ``machine1``, ``machine2``, and ``machine3``. You should use the actual machine names. If you have only 2 available machines, make ``machine2`` and ``machine3`` the same machine. Make sure each machines can talk to each other.
* Edit ``myconfig/clusters.xml``

  * Change the <machine> element for partition 0 from ``localhost`` to ``machine1``
  * Change the <machine> element for partition 1 from ``localhost`` to ``machine2``
  * Change the <machine> element for the adapter from ``localhost`` to ``machine3``
* Save your changes
* Copy your ``${IMAGE_BASE}`` directory to all three machines (but only those machines specified in the configuration).
* Start an S4 node on ``machine1`` (as above).
* Start an S4 node on ``machine2`` (as above).
* Start the adapter on ``machine3`` (as above).

You will need to check the log files on each machine to ensure the events are being distributed evenly.

Note: In the ``twittertopiccount`` application, only one node will write to the ``top_n_hashtags`` file.
