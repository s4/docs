.. index:: eclipse

Using Eclipse
=============

S4 is divided into 3 packages: ``core`` (Core classes), ``comm`` (Communication Layer), and ``examples`` (Example applications). Each of these packages corresponds to a github repository and can be imported into Eclipse as a separate project.

In this cookbook, we see how to start working on the S4 codebase using Eclipse.

1. Check out sources for the package that you need from `github <https://github.com/organizations/s4>`_.

  * Let's call the directory ``PKGDIR``

2. Create Eclipse configurations in ``PKGDIR``

  * ``cd PKGDIR``
  * ``mvn eclipse:eclipse``

3. Import project into Eclipse

  * :menuselection:`File --> Import --> General --> Existing Projects into Workspace`
  * Select ``PKGDIR`` from above as the "root directory" of the project.

4. Set variable ``M2_REPO`` to point to the local Maven repository

  * This is where Maven stores its local repository, e.g. ``~/.m2/repository``
  * Adding a variable in Eclipse: :menuselection:`Properties --> Java Build Path --> Libraries --> Add Variable`

5. Set up formatting:

  * Spaces for indentation
  * Tab width = 4