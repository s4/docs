S4 Documentation
================

This repository has sources for generating S4 documentation. 

S4 uses `reStructuredText <http://docutils.sourceforge.net/rst.html>`_ (RST) as documentation markup, and `Sphinx <http://sphinx.pocoo.org/>`_ as the engine for turning RST into HTML, PDF etc. 

Build Instructions
==================

Install Sphinx
--------------

Current build has been tested with Sphinx 1.0.5 on OS X 10.5. You will need a working Latex installation (eg, `Mactex <http://www.tug.org/mactex/>`_ on OS X) to generate PDF. 

``easy_install -U Sphinx``

Build
-----
* ``git clone git@github.com:s4/docs.git s4-docs``
* ``cd s4-docs``
* For HTML: ``make html``
* For PDF: ``make latexpdf``

