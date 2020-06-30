.. _gridconfiguration:

Grid configuration
==================

In some cases, the configuration properties will not suffice to fully
configure your server. In that case, it may be necessary to make use of
IceGrid's XML configuration files. Like the :file:`config.xml` file mentioned
above, these are stored under :file:`etc/grid`:
:file:`etc/grid/default.xml` is used on Unix systems and there is also
:file:`etc/grid/templates.xml`.

The default OMERO application descriptor deploys multiple server instances
(Blitz-0, FileServer, Indexer-0, PixelData-0, ...) on
a single node. Each server instance is defined by a ``server-template`` 
element in :file:`etc/grid/templates.xml` with its own parameters.

Modifying the application descriptors
-------------------------------------

When you run :program:`omero admin start` without any other arguments, it
looks up the default **application descriptor** for your platform:

::

	$ omero admin start
	No descriptor given. Using etc/grid/default.xml
	Waiting on startup. Use CTRL-C to exit

The "start" and "deploy" command, however, take several other
parameters:

::

    $ omero admin start --help
    usage: omero admin start [-h] [-u USER] [file] [targets [targets ...]]

    Start icegridnode daemon and waits for required components to come up,
    i.e. status == 0

    If the first argument can be found as a file, it will be deployed as the
    application descriptor rather than etc/grid/default.xml. All other
    arguments will be used as targets to enable optional sections of the
    descriptor

    Positional Arguments:
      file                  Application descriptor. If not provided, a default will be used
      targets               Targets within the application descriptor which should be activated.

If a file is passed in as the first argument, then that **application
descriptor** as opposed to :file:`etc/grid/default.xml` will
be used. You can also modify the default application descriptors in place.

.. note::
    The largest issue with using your own application
    descriptors or modifying the existing ones is that they tend to
    change between versions, and there is no facility for automatically
    merging your local changes. You should be prepared to re-make
    whatever changes you perform directly on the new files.

Targets
-------

**Targets** are elements within the application descriptors which can
optionally turn on configuration. The target is only applicable until
the next invocation of :program:`omero admin start` or :program:`omero admin deploy`

.. note::
    You must remember to always apply the targets on each
    :program:`omero admin` command. If not, the target will not be
    removed. Therefore, they are often better used for debugging purposes;
    however, as opposed to alternative application descriptors, using
    the pre-existing targets should not require any special effort
    during upgrades.

Debugging
^^^^^^^^^

::

    <properties id="PythonServer">
      <property name="Ice.ImplicitContext" value="Shared"/>
      <!-- Default logging settings for Python servers. -->
      <property name="omero.logging.timedlog" value="False"/>
      <property name="omero.logging.logsize" value="5000000"/>
      <property name="omero.logging.lognum" value="9"/>
      <property name="omero.logging.level" value="20"/>
      <target name="debug">
        <property name="omero.logging.level" value="10"/>
      </target>

Here, the "debug" target allows increasing the logging output of the
Python servers without modifying any files.

.. _jmx_configuration:

JMX configuration
^^^^^^^^^^^^^^^^^

::

    <server-template id="BlitzTemplate">
      <parameter name="index"/>
      <parameter name="config" default="default"/>
      <parameter name="jmxhost" default=""/>
      <parameter name="jmxport" default="3001"/>
       …
        <target name="jmx">
            <!-- Be sure to understand the consequences of enabling JMX.
                 It allows calling remote methods on your JVM -->
            <option>-Dcom.sun.management.jmxremote=${jmxhost}</option>
            <option>-Dcom.sun.management.jmxremote.port=${jmxport}</option>
            <option>-Dcom.sun.management.jmxremote.authenticate=false</option>
            <option>-Dcom.sun.management.jmxremote.ssl=false</option>
        </target>

The JMX target enables remote connections for external monitoring of the Blitz
server. If you need to modify the "jmxport" or "jmxhost" variables, you will
need to do so directly in the application descriptor XML.

Changing ports / multiple servers on a single host
--------------------------------------------------

By modifying the default OMERO ports, it is possible to run multiple
OMERO servers on the same physical machine. All port numbers can be adjusted
using the relevant :ref:`configuration properties <ports_configuration>`.

To run multiple servers on a single host, the easiest approach is to prefix all
ports (SSL, TCP, registry) using :property:`omero.ports.prefix`::

    # First server
    export OMERODIR=~/OMERO.server-1
    omero admin start

    # Second server
    export OMERODIR=~/OMERO.server-2
    omero config set omero.ports.prefix 1
    omero admin start

    # Third server
    export OMERODIR=~/OMERO.server-3
    omero config set omero.ports.prefix 2
    omero admin start


Clients will need to use the appropriate port (4064, 14064 or 24064) to
connect to OMERO.

.. seealso::

  :ref:`security_ssl`
    Section of the :doc:`/sysadmins/server-security` page.

Extending OMERO
---------------

Finally, if configuration does not suffice, there are also options for
extending OMERO with your own code. These are described on the
development site under |ExtendingOmero|.
