.. index:: maven, core, installation

Installing S4 core into your local maven repository
===================================================

There are two ways you can install the S4 core jar file into your local maven repository.

#. :doc:`Build S4 core </tutorials/build_core>` yourself.
#. Install the jar from your runnable image. Assuming the environmental variable ``IMAGE_BASE`` contains the base directory of your runnable image, do the following:

  * ``cd ${IMAGE_BASE}``
  * ``mvn install:install-file -DgroupId=io.s4 -DartifactId=s4_core -Dversion=0.2.1.0 -Dpackaging=jar -Dfile=s4_core/lib/s4_core-0.2.1.0.jar``
  * Note: If the version number on the jar file is not 0.2.1.0, adjust the ``-Dversion`` option and the jar file name accordingly.
