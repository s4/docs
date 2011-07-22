.. index:: application, gradle, maven

Building Your Application
==========================

Using gradle
------------

If you decide to use gradle to build your application, use the `twittertopiccount example <https://github.com/s4/twittertopiccount>`_ as a template.

If you use *twittertopiccount* as a template, the build process will rely on the value of the environmental variable $S4_IMAGE to find the S4-related jars (see :ref:`Set Up S4 <getting_started_set_up_application>` for instructions on setting that variable).

Setting up to use gradle
^^^^^^^^^^^^^^^^^^^^^^^^

Let's assume your application is located at <application base>.

  1. Download gradle from `here <http://www.gradle.org/>`_. Install.
  2. Copy the twittertopiccount's build.gradle file to <application base>.
  3. ``cd <application base>``
  4. ``mkdir -p src/main/resources``
  5. ``mkdir src/main/java``
  6. Edit :file:`<application base>/build.gradle`.

     1. Change the ``version`` and ``group`` variables appropriately.
     2. If this project includes a standalone client application (ala twitter_feed_listener), do the following:

        1. Update the ``mainClassName`` variable to contain the name of the main class of your client program.
        2. Update the ``applicationName`` variable to contain the name you wish for the wrapper script (gradle will generate the script for you).
     3. Change the list of dependencies to include those jars on which your application depends. The format of the entries is 'groupId:artifactId:version' in the Maven sense of those terms. Leave the first two entries (s4-core and s4-driver) in the list of dependencies.
     4. Change the entries in ``manifest.mainAttributes`` appropriately.

Building and deploying your project
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If this project includes a standalone client application (ala twitter_feed_listener), do the following:

   ``gradle install``

This will create a wrapper script called :file:`<application base>/build/install/<applicationName>/bin/<applicationName>`, where <applicationName> is the value you specified in the :file:`build.gradle` file.

To deploy the S4 application to $S4_IMAGE/s4-apps, do the following:

  ``gradle deploy``

To auto-generate eclipse project files, do the following

  ``gradle eclipse``

Unless you change dependencies, you need to do this only once.





