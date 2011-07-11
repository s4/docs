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
* ``git checkout tags/testtag``
* ``./gradlew allImage``
* ``cd build/s4-image/``
* ``export S4_IMAGE=`pwd```
* ``chmod u+x $S4_IMAGE/scripts/*``
* ``chmod u+x $S4_IMAGE/s4-driver/scripts/*``

If you want to build a specific release of S4, do the following before running ``./gradlew allImage``:

``git checkout tags/<tagname>``

Where <tagname> specifies the tag for a release, e.g. v0.3.0.

