.. index:: build, example, instructions

Build Sample Application
========================

This document assumes you have first followed the instructions in :doc:`building S4 core </tutorials/build_core>`.

The steps in this guide have been tested on:

* Red Hat Enterprise Linux 4
* Ubuntu 10.10

Build
-----

* Clone examples to a source directory (``<source_base>``)

  * ``cd <source_base>``
  * ``git clone git://github.com/s4/twittertopiccount.git``
* ``cd twittertopiccount``
* ``./gradlew install``
* Deploy the application: ``./gradlew deploy``
* ``cd build/install/twitter_feed_listener``
* ``export TWIT_LISTENER=`pwd```
