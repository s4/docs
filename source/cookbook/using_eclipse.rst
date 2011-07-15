.. index:: eclipse

Using Eclipse
=============

In this cookbook, we describe how to start working on the S4 codebase using Eclipse.

Using Eclipse to edit S4 codebase
----------------------------------

Formatting settings
^^^^^^^^^^^^^^^^^^^

When editing the S4 codebase, use the following formatting settings:

  * Spaces for indentation
  * Tab width = 4

Importing into eclipse
^^^^^^^^^^^^^^^^^^^^^^

  1. Clone the S4 repository and build according to :ref:`Set Up S4 <getting_started_set_up>`.
  2. Create eclipse configuration files: ``./gradlew eclipse``.
  3. From Eclipse, select file->import->Existing Projects into Workspace. Select the base of the newly cloned repository (e.g., :file:`<source base>/s4`) as the root directory.

This will import several subprojects into your workspace.
