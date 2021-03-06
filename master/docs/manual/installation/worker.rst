Worker Setup
============

.. _Creating-a-worker:

Creating a worker
-----------------

Typically, you will be adding a worker to an existing buildmaster, to provide additional architecture coverage.
The buildbot administrator will give you several pieces of information necessary to connect to the buildmaster.
You should also be somewhat familiar with the project being tested, so you can troubleshoot build problems locally.

The buildbot exists to make sure that the project's stated ``how to build it`` process actually works.
To this end, the worker should run in an environment just like that of your regular developers.
Typically the project build process is documented somewhere (:file:`README`, :file:`INSTALL`, etc), in a document that should mention all library dependencies and contain a basic set of build instructions.
This document will be useful as you configure the host and account in which the worker runs.

Here's a good checklist for setting up a worker:

1. Set up the account

  It is recommended (although not mandatory) to set up a separate user account for the worker.
  This account is frequently named ``buildbot`` or ``worker``.
  This serves to isolate your personal working environment from that of the worker's, and helps to minimize the security threat posed by letting possibly-unknown contributors run arbitrary code on your system.
  The account should have a minimum of fancy init scripts.

2. Install the buildbot code

  Follow the instructions given earlier (:ref:`Installing-the-code`).
  If you use a separate worker account, and you didn't install the buildbot code to a shared location, then you will need to install it with ``--home=~`` for each account that needs it.

3. Set up the host

  Make sure the host can actually reach the buildmaster.
  Usually the buildmaster is running a status webserver on the same machine, so simply point your web browser at it and see if you can get there.
  Install whatever additional packages or libraries the project's INSTALL document advises.
  (or not: if your worker is supposed to make sure that building without optional libraries still works, then don't install those libraries.)

  Again, these libraries don't necessarily have to be installed to a site-wide shared location, but they must be available to your build process.
  Accomplishing this is usually very specific to the build process, so installing them to :file:`/usr` or :file:`/usr/local` is usually the best approach.

4. Test the build process

  Follow the instructions in the :file:`INSTALL` document, in the worker's account.
  Perform a full CVS (or whatever) checkout, configure, make, run tests, etc.
  Confirm that the build works without manual fussing.
  If it doesn't work when you do it by hand, it will be unlikely to work when the buildbot attempts to do it in an automated fashion.

5. Choose a base directory

  This should be somewhere in the worker's account, typically named after the project which is being tested.
  The worker will not touch any file outside of this directory.
  Something like :file:`~/Buildbot` or :file:`~/Workers/fooproject` is appropriate.

6. Get the buildmaster host/port, botname, and password

  When the buildbot admin configures the buildmaster to accept and use your worker, they will provide you with the following pieces of information:

  * your worker's name
  * the password assigned to your worker
  * the hostname and port number of the buildmaster, i.e. `<http://buildbot.example.org:8007>`__

7. Create the worker

  Now run the 'worker' command as follows:

      :samp:`buildslave create-slave {BASEDIR} {MASTERHOST}:{PORT} {WORKERNAME} {PASSWORD}`

  This will create the base directory and a collection of files inside, including the :file:`buildbot.tac` file that contains all the information you passed to the :command:`buildbot` command.

8. Fill in the hostinfo files

  When it first connects, the worker will send a few files up to the buildmaster which describe the host that it is running on.
  These files are presented on the web status display so that developers have more information to reproduce any test failures that are witnessed by the buildbot.
  There are sample files in the :file:`info` subdirectory of the buildbot's base directory.
  You should edit these to correctly describe you and your host.

  :file:`{BASEDIR}/info/admin` should contain your name and email address.
  This is the ``buildslave admin address``, and will be visible from the build status page (so you may wish to munge it a bit if address-harvesting spambots are a concern).

  :file:`{BASEDIR}/info/host` should be filled with a brief description of the host: OS, version, memory size, CPU speed, versions of relevant libraries installed, and finally the version of the buildbot code which is running the worker.

  The optional :file:`{BASEDIR}/info/access_uri` can specify a URI which will connect a user to the machine.
  Many systems accept ``ssh://hostname`` URIs for this purpose.

  If you run many workers, you may want to create a single :file:`~worker/info` file and share it among all the workers with symlinks.

.. _Worker-Options:

Worker Options
~~~~~~~~~~~~~~

There are a handful of options you might want to use when creating the worker with the :samp:`buildslave create-slave <options> DIR <params>` command.
You can type ``buildslave create-slave --help`` for a summary.
To use these, just include them on the ``buildslave create-slave`` command line, like this

.. code-block:: bash

    buildslave create-slave --umask=022 ~/worker buildmaster.example.org:42012 {myworkername} {mypasswd}

.. program:: buildslave create-slave

.. option:: --no-logrotate

    This disables internal worker log management mechanism.
    With this option worker does not override the default logfile name and its behaviour giving a possibility to control those with command-line options of twistd daemon.

.. option:: --umask

    This is a string (generally an octal representation of an integer) which will cause the worker process' ``umask`` value to be set shortly after initialization.
    The ``twistd`` daemonization utility forces the umask to 077 at startup (which means that all files created by the worker or its child processes will be unreadable by any user other than the worker account).
    If you want build products to be readable by other accounts, you can add ``--umask=022`` to tell the worker to fix the umask after twistd clobbers it.
    If you want build products to be *writable* by other accounts too, use ``--umask=000``, but this is likely to be a security problem.

.. option:: --keepalive

    This is a number that indicates how frequently ``keepalive`` messages should be sent from the worker to the buildmaster, expressed in seconds.
    The default (600) causes a message to be sent to the buildmaster at least once every 10 minutes.
    To set this to a lower value, use e.g. ``--keepalive=120``.

    If the worker is behind a NAT box or stateful firewall, these messages may help to keep the connection alive: some NAT boxes tend to forget about a connection if it has not been used in a while.
    When this happens, the buildmaster will think that the worker has disappeared, and builds will time out.
    Meanwhile the worker will not realize than anything is wrong.

.. option:: --maxdelay

    This is a number that indicates the maximum amount of time the worker will wait between connection attempts, expressed in seconds.
    The default (300) causes the worker to wait at most 5 minutes before trying to connect to the buildmaster again.

.. option:: --log-size

    This is the size in bytes when to rotate the Twisted log files.

.. option:: --log-count

    This is the number of log rotations to keep around.
    You can either specify a number or ``None`` to keep all :file:`twistd.log` files around.
    The default is 10.

.. option:: --allow-shutdown

    Can also be passed directly to the BuildSlave constructor in :file:`buildbot.tac`.
    If set, it allows the worker to initiate a graceful shutdown, meaning that it will ask the master to shut down the worker when the current build, if any, is complete.

    Setting allow_shutdown to ``file`` will cause the worker to watch :file:`shutdown.stamp` in basedir for updates to its mtime.
    When the mtime changes, the worker will request a graceful shutdown from the master.
    The file does not need to exist prior to starting the worker.

    Setting allow_shutdown to ``signal`` will set up a SIGHUP handler to start a graceful shutdown.
    When the signal is received, the worker will request a graceful shutdown from the master.

    The default value is ``None``, in which case this feature will be disabled.

    Both master and worker must be at least version 0.8.3 for this feature to work.

.. _Other-Worker-Configuration:

Other Worker Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~

``unicode_encoding``
    This represents the encoding that buildbot should use when converting unicode commandline arguments into byte strings in order to pass to the operating system when spawning new processes.

    The default value is what Python's :func:`sys.getfilesystemencoding()` returns, which on Windows is 'mbcs', on Mac OSX is 'utf-8', and on Unix depends on your locale settings.

    If you need a different encoding, this can be changed in your worker's :file:`buildbot.tac` file by adding a ``unicode_encoding`` argument to the BuildSlave constructor.

.. code-block:: python

    s = Worker(buildmaster_host, port, workername, passwd, basedir,
               keepalive, usepty, umask=umask, maxdelay=maxdelay,
               unicode_encoding='utf-8', allow_shutdown='signal')

.. _Upgrading-an-Existing-Worker:

Upgrading an Existing Worker
----------------------------

If you have just installed a new version of Buildbot-slave, you may need to take some steps to upgrade it.
If you are upgrading to version 0.8.2 or later, you can run

.. code-block:: bash

    buildslave upgrade-slave /path/to/worker/dir

.. _Worker-Version-specific-Notes:

Version-specific Notes
~~~~~~~~~~~~~~~~~~~~~~

Upgrading a Worker to Buildbot-slave-0.8.1
''''''''''''''''''''''''''''''''''''''''''

Before Buildbot version 0.8.1, the Buildbot master and worker were part of the same distribution.
As of version 0.8.1, the worker is a separate distribution.

As of this release, you will need to install ``buildbot-slave`` to run a worker.

Any automatic startup scripts that had run ``buildbot start`` for previous versions should be changed to run ``buildslave start`` instead.

If you are running a version later than 0.8.1, then you can skip the remainder of this section: the ```upgrade-slave`` command will take care of this.
If you are upgrading directly to 0.8.1, read on.

The existing :file:`buildbot.tac` for any workers running older versions will need to be edited or replaced.
If the loss of cached worker state (e.g., for Source steps in copy mode) is not problematic, the easiest solution is to simply delete the worker directory and re-run ``buildslave create-slave``.

If deleting the worker directory is problematic, the change to :file:`buildbot.tac` is simple.
On line 3, replace::

    from buildbot.slave.bot import BuildSlave

with::

    from buildslave.bot import BuildSlave

After this change, the worker should start as usual.


