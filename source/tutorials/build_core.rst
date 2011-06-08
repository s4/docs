.. index:: build, core, instructions

Build Core
==========

The steps in this guide have been tested on:

* Red Hat Enterprise Linux 4
* Mac OSX 10.5

Build
-----

* Clone ``s4``to a source directory (``<source_base>``)

  * ``cd <source_base>``
  * ``git clone https://github.com/s4/s4.git``
* ``cd s4``
* ``./gradlew allImage``
* ``cd build/s4-image/``
* ``export S4_IMAGE=`pwd```

