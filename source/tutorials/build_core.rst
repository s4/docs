.. index:: build, core, instructions

Build Core
==========

The steps in this guide have been tested on:

* Red Hat Enterprise Linux 4
* Ubuntu 10.10

Build
-----

* Clone ``comm`` and ``core`` to a source directory (``<source_base>``)

  * ``cd <source_base>``
  * ``git clone https://github.com/s4/core.git``
  * ``git clone https://github.com/s4/comm.git``
* ``cd comm``
* Build (follow instructions in `README <https://github.com/s4/comm/blob/master/README.md>`_)
* ``cd ../core``
* Build (follow instructions in `README <https://github.com/s4/core/blob/master/README.md>`_)
* Locate the core tarball: ``ls -tl target/s4_core-*.tar.gz``

