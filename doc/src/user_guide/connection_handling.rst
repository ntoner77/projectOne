.. _connhandling:

*****************************
Connecting to Oracle Database
*****************************

This chapter covers connecting to Oracle Database using cx_Oracle.  It
explains the various forms of connections and how to manage them.

Establishing Database Connections
=================================

There are two ways to connect to Oracle Database using cx_Oracle:

*  **Standalone connections**

   These are useful when the application maintains a single user
   session to a database.  Connections are created by
   :meth:`cx_Oracle.connect()` or its alias
   :meth:`cx_Oracle.Connection()`.

*  **Pooled connections**

   Connection pooling is important for performance when applications
   frequently connect and disconnect from the database.  Oracle high
   availability features in the pool implementation mean that small
   pools can also be useful for applications that want a few
   connections available for infrequent use.  Pools are created with
   :meth:`cx_Oracle.SessionPool()` and then
   :meth:`SessionPool.acquire()` can be called to obtain a connection
   from a pool.

Optional connection creation parameters allow you to utilize features
such as Sharding and `Database Resident Connection Pooling (DRCP)`_.

Once a connection is established, you can use it for SQL, PL/SQL and
SODA.

**Example: Standalone Connection to Oracle Database**

.. code-block:: python

    import cx_Oracle

    userpwd = ". . ." # Obtain password string from a user prompt or environment variable

    connection = cx_Oracle.connect("hr", userpwd, "dbhost.example.com/orclpdb1", encoding="UTF-8")

cx_Oracle also supports :ref:`external authentication <extauth>` so
passwords do not need to be in the application.


Closing Connections
===================

Connections should be released when they are no longer needed by calling
:meth:`Connection.close()`.  Alternatively, you may prefer to let connections
be automatically cleaned up when references to them go out of scope.  This lets
cx_Oracle close dependent resources in the correct order. One other approach is
the use of a "with" block, which ensures that a connection is closed once the
block is completed. For example:

.. code-block:: python

    with cx_Oracle.connect(userName, password, "dbhost.example.com/orclpdb1",
                encoding="UTF-8") as connection:
        cursor = connection.cursor()
        cursor.execute("insert into SomeTable values (:1, :2)",
                (1, "Some string"))
        connection.commit()

This code ensures that, once the block is completed, the connection is closed
and resources have been reclaimed by the database. In addition, any attempt to
use the variable ``connection`` outside of the block will simply fail.


.. _envset:

Oracle Environment Variables
============================

Before running Python, ensure that any necessary Oracle environment
variables are configured correctly.  The variables needed by cx_Oracle
depend on how Python is installed, how you connect to the database,
and what optional settings are desired.

.. list-table:: Common Oracle environment variables
    :header-rows: 1
    :widths: 1 2
    :align: left

    * - Oracle Environment Variables
      - Purpose
    * - ORACLE_HOME
      - The directory containing the Oracle Database software. The directory
        and various configuration files must be readable by the Python process.
        This variable should not be set if you are using Oracle Instant Client.
    * - LD_LIBRARY_PATH
      - The library search path for platforms like Linux should include the
        Oracle libraries, for example ``$ORACLE_HOME/lib`` or
        ``/opt/instantclient_19_3``. This variable is not needed if the
        libraries are located by an alternative method, such as with
        ``ldconfig``. On other UNIX platforms you may need to set an OS
        specific equivalent, such as ``LIBPATH`` or ``SHLIB_PATH``.
    * - PATH
      - The library search path for Windows should include the location where
        ``OCI.DLL`` is found.
    * - TNS_ADMIN
      - The directory of Oracle Database client configuration files such as
        ``tnsnames.ora`` and ``sqlnet.ora``. Needed if the configuration files
        are in a non-default location.  See :ref:`optnetfiles`."
    * - NLS_LANG
      - Determines the 'national language support' globalization options for
        cx_Oracle. If not set, a default value will be chosen by Oracle. See
        :ref:`globalization`."
    * - NLS_DATE_FORMAT, NLS_TIMESTAMP_FORMAT
      - Often set in Python applications to force a consistent date format
        independent of the locale. The variables are ignored if the environment
        variable ``NLS_LANG`` is not set.

It is recommended to set Oracle variables in the environment before
invoking Python. However, they may also be set in application code with
``os.putenv()`` before the first connection is established.  Note that setting
operating system variables such as ``LD_LIBRARY_PATH`` must be done
before running Python.


Optional Oracle Configuration Files
===================================

.. _optnetfiles:

Optional Oracle Net Configuration Files
---------------------------------------

Optional Oracle Net configuration files affect connections and
applications.

Common files include:

* ``tnsnames.ora``: A configuration file that defines databases addresses
  for establishing connections. See :ref:`Net Service Name for Connection
  Strings <netservice>`.

* ``sqlnet.ora``: A profile configuration file that may contain information
  on features such as connection failover, network encryption, logging, and
  tracing.  See `Oracle Net Services Reference
  <https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
  id=GUID-19423B71-3F6C-430F-84CC-18145CC2A818>`__ for more information.

* ``cwallet.sso``: an Oracle wallet for secure connection.

The default location for these files is the ``network/admin``
directory under the Oracle Instant Client installation directory or the
``$ORACLE_HOME`` directory (for full database or client installations). To use
a non-default location, put the files in a directory that is accessible to
Python and set the ``TNS_ADMIN`` environment variable to
that directory path.  For example, if the file
``/etc/my-oracle-config/tnsnames.ora`` is being used, set the
``TNS_ADMIN`` environment variable to ``/etc/my-oracle-config``.

.. _optclientfiles:

Optional Oracle Client Configuration Files
------------------------------------------

When cx_Oracle uses Oracle Database Clients 12.1, or later, an optional client
parameter file called ``oraaccess.xml`` can be used.  This file can be used to
override some application settings, which can be useful if the application
cannot be altered.  The file also enables auto-tuning of the client statement
cache.

The file is read from the same directory as the
`Optional Oracle Net Configuration Files`_.

A sample ``oraaccess.xml`` file that sets the Oracle client ‘prefetch’
value to 50 rows and the 'client statement cache' value to 1, is shown
below::

    <oraaccess xmlns="http://xmlns.oracle.com/oci/oraaccess"
            xmlns:oci="http://xmlns.oracle.com/oci/oraaccess"
            schemaLocation="http://xmlns.oracle.com/oci/oraaccess
            http://xmlns.oracle.com/oci/oraaccess.xsd">
        <default_parameters>
            <prefetch>
                <rows>50</rows>
            </prefetch>
            <statement_cache>
                <size>1</size>
            </statement_cache>
        </default_parameters>
    </oraaccess>

Refer to the documentation on `oraaccess.xml
<https://www.oracle.com/pls/topic/lookup?
ctx=dblatest&id=GUID-9D12F489-EC02-46BE-8CD4-5AECED0E2BA2>`__
for more details.

.. _connstr:

Connection Strings
==================

The data source name parameter ``dsn`` of :meth:`cx_Oracle.connect()` and
:meth:`cx_Oracle.SessionPool()` is the Oracle Database connection string
identifying which database service to connect to. The ``dsn`` string can be one
of:

* An Oracle Easy Connect string
* An Oracle Net Connect Descriptor string
* A Net Service Name mapping to a connect descriptor

For more information about naming methods, see `Oracle Net Service Reference <https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-E5358DEA-D619-4B7B-A799-3D2F802500F1>`__.

.. _easyconnect:

Easy Connect Syntax for Connection Strings
------------------------------------------

An Easy Connect string is often the simplest connection string to use for the
data source name parameter ``dsn`` of :meth:`cx_Oracle.connect()` and
:meth:`cx_Oracle.SessionPool()`.  This method does not need configuration files
such as ``tnsnames.ora``.

For example, to connect to the Oracle Database service ``orclpdb1`` that is
running on the host ``dbhost.example.com`` with the default Oracle
Database port 1521, use::

    connection = cx_Oracle.connect("hr", userpwd, "dbhost.example.com/orclpdb1",
            encoding="UTF-8")

If the database is using a non-default port, it must be specified::

    connection = cx_Oracle.connect("hr", userpwd, "dbhost.example.com:1984/orclpdb1",
            encoding="UTF-8")

The Easy Connect syntax supports Oracle Database service names.  It cannot be
used with the older System Identifiers (SID).

The Easy Connect syntax has been extended in recent versions of Oracle Database
client since its introduction in 10g.  Check the Easy Connect Naming method in
`Oracle Net Service Administrator's Guide
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-B0437826-43C1-49EC-A94D-B650B6A4A6EE>`__ for the syntax to use in your
version of the Oracle Client libraries.

If you are using Oracle Client 19c, the latest `Easy Connect Plus
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-8C85D289-6AF3-41BC-848B-BF39D32648BA>`__ syntax allows the use of
multiple hosts or ports, along with optional entries for the wallet location,
the distinguished name of the database server, and even lets some network
configuration options be set. This means that a :ref:`sqlnet.ora <optnetfiles>`
file is not needed for some common connection scenarios.

Oracle Net Connect Descriptor Strings
-------------------------------------

The :meth:`cx_Oracle.makedsn()` function can be used to construct a connect
descriptor string for the data source name parameter ``dsn`` of
:meth:`cx_Oracle.connect()` and :meth:`cx_Oracle.SessionPool()`.  The
:meth:`~cx_Oracle.makedsn()` function accepts the database hostname, the port
number, and the service name.  It also supports :ref:`sharding <connsharding>`
syntax.

For example, to connect to the Oracle Database service ``orclpdb1`` that is
running on the host ``dbhost.example.com`` with the default Oracle
Database port 1521, use::

    dsn = cx_Oracle.makedsn("dbhost.example.com", 1521, service_name="orclpdb1")
    connection = cx_Oracle.connect("hr", userpwd, dsn, encoding="UTF-8")

Note the use of the named argument ``service_name``.  By default, the third
parameter of :meth:`~cx_Oracle.makedsn()` is a database System Identifier (SID),
not a service name.  However, almost all current databases use service names.

The value of ``dsn`` in this example is the connect descriptor string::

    (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=dbhost.example.com)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=orclpdb1)))

You can manually create similar connect descriptor strings.  This lets you
extend the syntax, for example to support failover.  These strings can be
embedded directly in the application::

    dsn = """(DESCRIPTION=
                 (FAILOVER=on)
                 (ADDRESS_LIST=
                   (ADDRESS=(PROTOCOL=tcp)(HOST=sales1-svr)(PORT=1521))
                   (ADDRESS=(PROTOCOL=tcp)(HOST=sales2-svr)(PORT=1521)))
                 (CONNECT_DATA=(SERVICE_NAME=sales.example.com)))"""

    connection = cx_Oracle.connect("hr", userpwd, dsn, encoding="UTF-8")

.. _netservice:

Net Service Names for Connection Strings
----------------------------------------

Connect Descriptor Strings are commonly stored in a :ref:`tnsnames.ora
<optnetfiles>` file and associated with a Net Service Name.  This name can be
used directly for the data source name parameter ``dsn`` of
:meth:`cx_Oracle.connect()` and :meth:`cx_Oracle.SessionPool()`.  For example,
given a ``tnsnames.ora`` file with the following contents::

    ORCLPDB1 =
      (DESCRIPTION =
        (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost.example.com)(PORT = 1521))
        (CONNECT_DATA =
          (SERVER = DEDICATED)
          (SERVICE_NAME = orclpdb1)
        )
      )

then you could connect using the following code::

    connection = cx_Oracle.connect("hr", userpwd, "orclpdb1", encoding="UTF-8")

For more information about Net Service Names, see
`Database Net Services Reference
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-12C94B15-2CE1-4B98-9D0C-8226A9DDF4CB>`__.

JDBC and Oracle SQL Developer Connection Strings
------------------------------------------------

The cx_Oracle connection string syntax is different to Java JDBC and the common
Oracle SQL Developer syntax.  If these JDBC connection strings reference a
service name like::

    jdbc:oracle:thin:@hostname:port/service_name

for example::

    jdbc:oracle:thin:@dbhost.example.com:1521/orclpdb1

then use Oracle's Easy Connect syntax in cx_Oracle::

    connection = cx_Oracle.connect("hr", userpwd, "dbhost.example.com:1521/orclpdb1", encoding="UTF-8")

Alternatively, if a JDBC connection string uses an old-style Oracle SID "system
identifier", and the database does not have a service name::

    jdbc:oracle:thin:@hostname:port:sid

for example::

    jdbc:oracle:thin:@dbhost.example.com:1521:orcl

then a connect descriptor string from ``makedsn()`` can be used in the
application::

    dsn = cx_Oracle.makedsn("dbhost.example.com", 1521, sid="orcl")
    connection = cx_Oracle.connect("hr", userpwd, dsn, encoding="UTF-8")

Alternatively, create a ``tnsnames.ora`` (see :ref:`optnetfiles`) entry, for
example::

    finance =
     (DESCRIPTION =
       (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost.example.com)(PORT = 1521))
       (CONNECT_DATA =
         (SID = ORCL)
       )
     )

This can be referenced in cx_Oracle::

    connection = cx_Oracle.connect("hr", userpwd, "finance", encoding="UTF-8")

.. _connpool:

Connection Pooling
==================

cx_Oracle's connection pooling lets applications create and maintain a pool of
connections to the database.  The internal implementation uses Oracle's
`session pool technology <https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-F9662FFB-EAEF-495C-96FC-49C6D1D9625C>`__.
In general, each connection in a cx_Oracle connection pool corresponds to one
Oracle session.

A connection pool is created by calling :meth:`~cx_Oracle.SessionPool()`.  This
is generally called during application initialization.  Connections can then be
obtained from a pool by calling :meth:`~SessionPool.acquire()`.  The initial
pool size and the maximum pool size are provided at the time of pool creation.
When the pool needs to grow, new connections are created automatically.  The
pool can shrink back to the minimum size when connections are no longer in use.
See `Connection Pooling`_ for more information.

Connections acquired from the pool should be released back to the pool using
:meth:`SessionPool.release()` or :meth:`Connection.close()` when they are no
longer required.  Otherwise, they will be released back to the pool
automatically when all of the variables referencing the connection go out of
scope.  The session pool can be completely closed using
:meth:`SessionPool.close()`.

The example below shows how to connect to Oracle Database using a
connection pool:

.. code-block:: python

    # Create the session pool
    pool = cx_Oracle.SessionPool("hr", userpwd,
            "dbhost.example.com/orclpdb1", min=2, max=5, increment=1, encoding="UTF-8")

    # Acquire a connection from the pool
    connection = pool.acquire()

    # Use the pooled connection
    cursor = connection.cursor()
    for result in cursor.execute("select * from mytab"):
        print(result)

    # Release the connection to the pool
    pool.release(connection)

    # Close the pool
    pool.close()

Applications that are using connections concurrently in multiple threads should
set the ``threaded`` parameter to True when creating a connection pool:

.. code-block:: python

    # Create the session pool
    pool = cx_Oracle.SessionPool("hr", userpwd, "dbhost.example.com/orclpdb1",
                  min=2, max=5, increment=1, threaded=True, encoding="UTF-8")

See `Threads.py
<https://github.com/oracle/python-cx_Oracle/tree/master/samples/Threads.py>`__
for an example.

The Oracle Real-World Performance Group's general recommendation for connection
pools is use a fixed sized pool.  The values of `min` and `max` should be the
same (and `increment` equal to zero).  the firewall, `resource manager
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-2BEF5482-CF97-4A85-BD90-9195E41E74EF>`__
or user profile `IDLE_TIME
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-ABC7AE4D-64A8-4EA9-857D-BEF7300B64C3>`__
should not expire idle sessions.  This avoids connection storms which can
decrease throughput.  See `About Optimizing Real-World Performance with Static
Connection Pools
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-BC09F045-5D80-4AF5-93F5-FEF0531E0E1D>`__,
which contains details about sizing of pools.

Session CallBacks for Setting Pooled Connection State
-----------------------------------------------------

Applications can set "session" state in each connection.  Examples of session
state are NLS settings from ``ALTER SESSION`` statements.  Pooled connections
will retain their session state after they have been released back to the pool.
However, because pools can grow, or connections in the pool can be recreated,
there is no guarantee a subsequent :meth:`~SessionPool.acquire()` call will
return a database connection that has any particular state.

The :meth:`~cx_Oracle.SessionPool()` parameter ``sessionCallback``
enables efficient setting of session state so that connections have a
known session state, without requiring that state to be explicitly set
after each :meth:`~SessionPool.acquire()` call.

Connections can also be tagged when they are released back to the pool.  The
tag is a user-defined string that represents the session state of the
connection.  When acquiring connections, a particular tag can be requested.  If
a connection with that tag is available, it will be returned.  If not, then
another session will be returned.  By comparing the actual and requested tags,
applications can determine what exact state a session has, and make any
necessary changes.

The session callback can be a Python function or a PL/SQL procedure.

There are three common scenarios for ``sessionCallback``:

- When all connections in the pool should have the same state, use a
  Python callback without tagging.

- When connections in the pool require different state for different
  users, use a Python callback with tagging.

- When using :ref:`drcp`: use a PL/SQL callback with tagging.


**Python Callback**

If the ``sessionCallback`` parameter is a Python procedure, it will be called
whenever :meth:`~SessionPool.acquire()` will return a newly created database
connection that has not been used before.  It is also called when connection
tagging is being used and the requested tag is not identical to the tag in the
connection returned by the pool.

An example is:

.. code-block:: python

    # Set the NLS_DATE_FORMAT for a session
    def initSession(connection, requestedTag):
        cursor = connection.cursor()
        cursor.execute("ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD HH24:MI'")

    # Create the pool with session callback defined
    pool = cx_Oracle.SessionPool("hr", userpwd, "orclpdb1",
                         sessionCallback=initSession, encoding="UTF-8")

    # Acquire a connection from the pool (will always have the new date format)
    connection = pool.acquire()

If needed, the ``initSession()`` procedure is called internally before
``acquire()`` returns.  It will not be called when previously used connections
are returned from the pool.  This means that the ALTER SESSION does not need to
be executed after every ``acquire()`` call.  This improves performance and
scalability.

In this example tagging was not being used, so the ``requestedTag`` parameter
is ignored.

**Connection Tagging**

Connection tagging is used when connections in a pool should have differing
session states.  In order to retrieve a connection with a desired state, the
``tag`` attribute in :meth:`~SessionPool.acquire()` needs to be set.

When cx_Oracle is using Oracle Client libraries 12.2 or later, then cx_Oracle
uses 'multi-property tags' and the tag string must be of the form of one or
more "name=value" pairs separated by a semi-colon, for example
``"loc=uk;lang=cy"``.

When a connection is requested with a given tag, and a connection with that tag
is not present in the pool, then a new connection, or an existing connection
with cleaned session state, will be chosen by the pool and the session callback
procedure will be invoked.  The callback can then set desired session state and
update the connection's tag.  However if the ``matchanytag`` parameter of
:meth:`~SessionPool.acquire()` is *True*, then any other tagged connection may
be chosen by the pool and the callback procedure should parse the actual and
requested tags to determine which bits of session state should be reset.

The example below demonstrates connection tagging:

.. code-block:: python

    def initSession(connection, requestedTag):
        if requestedTag == "NLS_DATE_FORMAT=SIMPLE":
            sql = "ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD'"
        elif requestedTag == "NLS_DATE_FORMAT=FULL":
            sql = "ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD HH24:MI'"
        cursor = connection.cursor()
        cursor.execute(sql)
        connection.tag = requestedTag

    pool = cx_Oracle.SessionPool("hr", userpwd, "orclpdb1",
                         sessionCallback=initSession, encoding="UTF-8")

    # Two connections with different session state:
    connection1 = pool.acquire(tag = "NLS_DATE_FORMAT=SIMPLE")
    connection2 = pool.acquire(tag = "NLS_DATE_FORMAT=FULL")

See `SessionCallback.py
<https://github.com/oracle/python-cx_Oracle/tree/master/
samples/SessionCallback.py>`__ for an example.

**PL/SQL Callback**

When cx_Oracle uses Oracle Client 12.2 or later, the session callback can also
be the name of a PL/SQL procedure.  A PL/SQL callback will be initiated only
when the tag currently associated with a connection does not match the tag that
is requested.  A PL/SQL callback is most useful when using :ref:`drcp` because
DRCP does not require a round-trip to invoke a PL/SQL session callback
procedure.

The PL/SQL session callback should accept two VARCHAR2 arguments:

.. code-block:: sql

    PROCEDURE myPlsqlCallback (
        requestedTag IN  VARCHAR2,
        actualTag    IN  VARCHAR2
    );

The logic in this procedure can parse the actual tag in the session that has
been selected by the pool and compare it with the tag requested by the
application.  The procedure can then change any state required before the
connection is returned to the application from :meth:`~SessionPool.acquire()`.

If the ``matchanytag`` attribute of :meth:`~SessionPool.acquire()` is *True*,
then a connection with any state may be chosen by the pool.

Oracle 'multi-property tags' must be used.  The tag string must be of the form
of one or more "name=value" pairs separated by a semi-colon, for example
``"loc=uk;lang=cy"``.

In cx_Oracle set ``sessionCallback`` to the name of the PL/SQL procedure. For
example:

.. code-block:: python

    pool = cx_Oracle.SessionPool("hr", userpwd, "dbhost.example.com/orclpdb1:pooled",
                         sessionCallback="myPlsqlCallback", encoding="UTF-8")

    connection = pool.acquire(tag="NLS_DATE_FORMAT=SIMPLE",
            # DRCP options, if you are using DRCP
            cclass='MYCLASS', purity=cx_Oracle.ATTR_PURITY_SELF)

See `SessionCallbackPLSQL.py
<https://github.com/oracle/python-cx_Oracle/tree/master/
samples/SessionCallbackPLSQL.py>`__ for an example.

.. _connpooltypes:

Heterogeneous and Homogeneous Connection Pools
----------------------------------------------

By default, connection pools are ‘homogeneous’, meaning that all connections
use the same database credentials.  However, if the pool option ``homogeneous``
is False at the time of pool creation, then a ‘heterogeneous’ pool will be
created.  This allows different credentials to be used each time a connection
is acquired from the pool with :meth:`~SessionPool.acquire()`.

**Heterogeneous Pools**

When a heterogeneous pool is created by setting ``homogeneous`` to False and no
credentials are supplied during pool creation, then a user name and password
may be passed to :meth:`~SessionPool.acquire()` as shown in this example:

.. code-block:: python

    pool = cx_Oracle.SessionPool(dsn="dbhost.example.com/orclpdb1", homogeneous=False,
                         encoding="UTF-8")
    connection = pool.acquire(user="hr", password=userpwd)

.. _drcp:

Database Resident Connection Pooling (DRCP)
===========================================

`Database Resident Connection Pooling (DRCP)
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-015CA8C1-2386-4626-855D-CC546DDC1086>`__ enables database resource
sharing for applications that run in multiple client processes, or run on
multiple middle-tier application servers.  By default each connection from
Python will use one database server process.  DRCP allows pooling of these
server processes.  This reduces the amount of memory required on the database
host.  The DRCP pool can be shared by multiple applications.

DRCP is useful for applications which share the same database credentials, have
similar session settings (for example date format settings or PL/SQL package
state), and where the application gets a database connection, works on it for a
relatively short duration, and then releases it.

Applications can choose whether or not to use pooled connections at runtime.

For efficiency, it is recommended that DRCP connections should be used
in conjunction with cx_Oracle’s local :ref:`connection pool <connpool>`.

**Using DRCP in Python**

Using DRCP with cx_Oracle applications involves the following steps:

1. Configuring and enabling DRCP in the database
2. Configuring the application to use a DRCP connection
3. Deploying the application

**Configuring and enabling DRCP**

Every instance of Oracle Database uses a single, default connection
pool. The pool can be configured and administered by a DBA using the
``DBMS_CONNECTION_POOL`` package:

.. code-block:: sql

    EXECUTE DBMS_CONNECTION_POOL.CONFIGURE_POOL(
        pool_name => 'SYS_DEFAULT_CONNECTION_POOL',
        minsize => 4,
        maxsize => 40,
        incrsize => 2,
        session_cached_cursors => 20,
        inactivity_timeout => 300,
        max_think_time => 600,
        max_use_session => 500000,
        max_lifetime_session => 86400)

Alternatively the method ``DBMS_CONNECTION_POOL.ALTER_PARAM()`` can
set a single parameter:

.. code-block:: sql

    EXECUTE DBMS_CONNECTION_POOL.ALTER_PARAM(
        pool_name => 'SYS_DEFAULT_CONNECTION_POOL',
        param_name => 'MAX_THINK_TIME',
        param_value => '1200')

The ``inactivity_timeout`` setting terminates idle pooled servers, helping
optimize database resources.  To avoid pooled servers permanently being held
onto by a selfish Python script, the ``max_think_time`` parameter can be set.
The parameters ``num_cbrok`` and ``maxconn_cbrok`` can be used to distribute
the persistent connections from the clients across multiple brokers.  This may
be needed in cases where the operating system per-process descriptor limit is
small.  Some customers have found that having several connection brokers
improves performance.  The ``max_use_session`` and ``max_lifetime_session``
parameters help protect against any unforeseen problems affecting server
processes.  The default values will be suitable for most users.  See the
`Oracle DRCP documentation
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-015CA8C1-2386-4626-855D-CC546DDC1086>`__ for details on parameters.

In general, if pool parameters are changed, the pool should be restarted,
otherwise server processes will continue to use old settings.

There is a ``DBMS_CONNECTION_POOL.RESTORE_DEFAULTS()`` procedure to
reset all values.

When DRCP is used with RAC, each database instance has its own connection
broker and pool of servers.  Each pool has the identical configuration.  For
example, all pools start with ``minsize`` server processes.  A single
DBMS_CONNECTION_POOL command will alter the pool of each instance at the same
time.  The pool needs to be started before connection requests begin.  The
command below does this by bringing up the broker, which registers itself with
the database listener:

.. code-block:: sql

    EXECUTE DBMS_CONNECTION_POOL.START_POOL()

Once enabled this way, the pool automatically restarts when the database
instance restarts, unless explicitly stopped with the
``DBMS_CONNECTION_POOL.STOP_POOL()`` command:

.. code-block:: sql

    EXECUTE DBMS_CONNECTION_POOL.STOP_POOL()

The pool cannot be stopped while connections are open.

**Application Deployment for DRCP**

In order to use DRCP, the ``cclass`` and ``purity`` parameters should
be passed to :meth:`cx_Oracle.connect()` or :meth:`SessionPool.acquire()`.  If
``cclass`` is not set, the pooled server sessions will not be reused optimally,
and the DRCP statistic views will record large values for NUM_MISSES.

The DRCP ``purity`` can be one of ``ATTR_PURITY_NEW``, ``ATTR_PURITY_SELF``,
or ``ATTR_PURITY_DEFAULT``.  The value ``ATTR_PURITY_SELF`` allows reuse of
both the pooled server process and session memory, giving maximum benefit from
DRCP.  See the Oracle documentation on `benefiting from scalability
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-661BB906-74D2-4C5D-9C7E-2798F76501B3>`__.

The connection string used for :meth:`~cx_Oracle.connect()` or
:meth:`~SessionPool.acquire()` must request a pooled server by
following one of the syntaxes shown below:

Using Oracle’s Easy Connect syntax, the connection would look like:

.. code-block:: python

    connection = cx_Oracle.connect("hr", userpwd, "dbhost.example.com/orcl:pooled",
            encoding="UTF-8")

Or if you connect using a Net Service Name named ``customerpool``:

.. code-block:: python

    connection = cx_Oracle.connect("hr", userpwd, "customerpool", encoding="UTF-8")

Then only the Oracle Network configuration file ``tnsnames.ora`` needs
to be modified::

    customerpool = (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)
              (HOST=dbhost.example.com)
              (PORT=1521))(CONNECT_DATA=(SERVICE_NAME=CUSTOMER)
              (SERVER=POOLED)))

If these changes are made and the database is not actually configured for DRCP,
or the pool is not started, then connections will not succeed and an error will
be returned to the Python application.

Although applications can choose whether or not to use pooled connections at
runtime, care must be taken to configure the database appropriately for the
number of expected connections, and also to stop inadvertent use of non-DRCP
connections leading to a resource shortage.

The example below shows how to connect to Oracle Database using Database
Resident Connection Pooling:

.. code-block:: python

    connection = cx_Oracle.connect("hr", userpwd, "dbhost.example.com/orcl:pooled",
            cclass="MYCLASS", purity=cx_Oracle.ATTR_PURITY_SELF, encoding="UTF-8")

The example below shows connecting to Oracle Database using DRCP and
cx_Oracle's connection pooling:

.. code-block:: python

    mypool = cx_Oracle.SessionPool("hr", userpwd, "dbhost.example.com/orcl:pooled",
                           encoding="UTF-8")
    connection = mypool.acquire(cclass="MYCLASS", purity=cx_Oracle.ATTR_PURITY_SELF)

For more information about DRCP see `Oracle Database Concepts Guide
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-531EEE8A-B00A-4C03-A2ED-D45D92B3F797>`__, and for DRCP Configuration
see `Oracle Database Administrator's Guide
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-82FF6896-F57E-41CF-89F7-755F3BC9C924>`__.

**Closing Connections**

Python scripts where cx_Oracle connections do not go out of scope quickly
(which releases them), or do not currently use :meth:`Connection.close()`,
should be examined to see if :meth:`~Connection.close()` can be used, which
then allows maximum use of DRCP pooled servers by the database:

.. code-block:: python

     # Do some database operations
    connection = cx_Oracle.connect("hr", userpwd, "dbhost.example.com/orclpdb1:pooled",
            encoding="UTF-8")
    . . .
    connection.close();

    # Do lots of non-database work
    . . .

    # Do some more database operations
    connection = cx_Oracle.connect("hr", userpwd, "dbhost.example.com/orclpdb1:pooled",
            encoding="UTF-8")
    . . .
    connection.close();

**Monitoring DRCP**

Data dictionary views are available to monitor the performance of DRCP.
Database administrators can check statistics such as the number of busy and
free servers, and the number of hits and misses in the pool against the total
number of requests from clients. The views are:

* ``DBA_CPOOL_INFO``
* ``V$PROCESS``
* ``V$SESSION``
* ``V$CPOOL_STATS``
* ``V$CPOOL_CC_STATS``
* ``V$CPOOL_CONN_INFO``

**DBA_CPOOL_INFO View**

``DBA_CPOOL_INFO`` displays configuration information about the DRCP pool.  The
columns are equivalent to the ``dbms_connection_pool.configure_pool()``
settings described in the table of DRCP configuration options, with the
addition of a ``STATUS`` column.  The status is ``ACTIVE`` if the pool has been
started and ``INACTIVE`` otherwise.  Note the pool name column is called
``CONNECTION_POOL``.  This example checks whether the pool has been started and
finds the maximum number of pooled servers::

    SQL> SELECT connection_pool, status, maxsize FROM dba_cpool_info;

    CONNECTION_POOL              STATUS        MAXSIZE
    ---------------------------- ---------- ----------
    SYS_DEFAULT_CONNECTION_POOL  ACTIVE             40

**V$PROCESS and V$SESSION Views**

The ``V$SESSION`` view shows information about the currently active DRCP
sessions.  It can also be joined with ``V$PROCESS`` via
``V$SESSION.PADDR = V$PROCESS.ADDR`` to correlate the views.

**V$CPOOL_STATS View**

The ``V$CPOOL_STATS`` view displays information about the DRCP statistics for
an instance.  The V$CPOOL_STATS view can be used to assess how efficient the
pool settings are. T his example query shows an application using the pool
effectively.  The low number of misses indicates that servers and sessions were
reused.  The wait count shows just over 1% of requests had to wait for a pooled
server to become available::

    NUM_REQUESTS   NUM_HITS NUM_MISSES  NUM_WAITS
    ------------ ---------- ---------- ----------
           10031      99990         40       1055

If ``cclass`` was set (allowing pooled servers and sessions to be
reused) then NUM_MISSES will be low.  If the pool maxsize is too small for
the connection load, then NUM_WAITS will be high.

**V$CPOOL_CC_STATS View**

The view ``V$CPOOL_CC_STATS`` displays information about the connection class
level statistics for the pool per instance::

    SQL> SELECT cclass_name, num_requests, num_hits, num_misses
         FROM v$cpool_cc_stats;

    CCLASS_NAME                      NUM_REQUESTS NUM_HITS   NUM_MISSES
    -------------------------------- ------------ ---------- ----------
    HR.MYCLASS                             100031      99993         38

**V$CPOOL_CONN_INFO View**

The ``V$POOL_CONN_INFO`` view gives insight into client processes that are
connected to the connection broker, making it easier to monitor and trace
applications that are currently using pooled servers or are idle. This view was
introduced in Oracle 11gR2.

You can monitor the view ``V$CPOOL_CONN_INFO`` to, for example, identify
misconfigured machines that do not have the connection class set correctly.
This view maps the machine name to the class name::

    SQL> SELECT cclass_name, machine FROM v$cpool_conn_info;

    CCLASS_NAME                             MACHINE
    --------------------------------------- ------------
    CJ.OCI:SP:wshbIFDtb7rgQwMyuYvodA        cjlinux
    . . .

In this example you would examine applications on ``cjlinux`` and make
sure ``cclass`` is set.


.. _proxyauth:

Connecting Using Proxy Authentication
=====================================

Proxy authentication allows a user (the "session user") to connect to Oracle
Database using the credentials of a 'proxy user'.  Statements will run as the
session user.  Proxy authentication is generally used in three-tier applications
where one user owns the schema while multiple end-users access the data.  For
more information about proxy authentication, see the `Oracle documentation
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-D77D0D4A-7483-423A-9767-CBB5854A15CC>`__.

An alternative to using proxy users is to set
:attr:`Connection.client_identifier` after connecting and use its value in
statements and in the database, for example for :ref:`monitoring
<endtoendtracing>`.

The following proxy examples use these schemas.  The ``mysessionuser`` schema is
granted access to use the password of ``myproxyuser``:

.. code-block:: sql

    CREATE USER myproxyuser IDENTIFIED BY myproxyuserpw;
    GRANT CREATE SESSION TO myproxyuser;

    CREATE USER mysessionuser IDENTIFIED BY itdoesntmatter;
    GRANT CREATE SESSION TO mysessionuser;

    ALTER USER mysessionuser GRANT CONNECT THROUGH myproxyuser;

After connecting to the database, the following query can be used to show the
session and proxy users:

.. code-block:: sql

    SELECT SYS_CONTEXT('USERENV', 'PROXY_USER'),
           SYS_CONTEXT('USERENV', 'SESSION_USER')
    FROM DUAL;

Standalone connection examples:

.. code-block:: python

    # Basic Authentication without a proxy
    connection = cx_Oracle.connect("myproxyuser", "myproxyuserpw", "dbhost.example.com/orclpdb1",
            encoding="UTF-8")
    # PROXY_USER:   None
    # SESSION_USER: MYPROXYUSER

    # Basic Authentication with a proxy
    connection = cx_Oracle.connect(user="myproxyuser[mysessionuser]", "myproxyuserpw",
           "dbhost.example.com/orclpdb1", encoding="UTF-8")
    # PROXY_USER:   MYPROXYUSER
    # SESSION_USER: MYSESSIONUSER

Pooled connection examples:

.. code-block:: python

    # Basic Authentication without a proxy
    pool = cx_Oracle.SessionPool("myproxyuser", "myproxyuser", "dbhost.example.com/orclpdb1",
                         encoding="UTF-8")
    connection = pool.acquire()
    # PROXY_USER:   None
    # SESSION_USER: MYPROXYUSER

    # Basic Authentication with proxy
    pool = cx_Oracle.SessionPool("myproxyuser[mysessionuser]", "myproxyuser",
                         "dbhost.example.com/orclpdb1", homogeneous=False, encoding="UTF-8")
    connection = pool.acquire()
    # PROXY_USER:   MYPROXYUSER
    # SESSION_USER: MYSESSIONUSER

Note the use of a :ref:`heterogeneous <connpooltypes>` pool in the example
above.  This is required in this scenario.

.. _extauth:

Connecting Using External Authentication
========================================

Instead of storing the database username and password in Python scripts or
environment variables, database access can be authenticated by an outside
system.  External Authentication allows applications to validate user access by
an external password store (such as an Oracle Wallet), by the operating system,
or with an external authentication service.

Using an Oracle Wallet for External Authentication
--------------------------------------------------

The following steps give an overview of using an Oracle Wallet.  Wallets should
be kept securely.  Wallets can be managed with `Oracle Wallet Manager
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-E3E16C82-E174-4814-98D5-EADF1BCB3C37>`__.

In this example the wallet is created for the ``myuser`` schema in the directory
``/home/oracle/wallet_dir``.  The ``mkstore`` command is available from a full
Oracle client or Oracle Database installation.  If you have been given wallet by
your DBA, skip to step 3.

1.  First create a new wallet as the ``oracle`` user::

        mkstore -wrl "/home/oracle/wallet_dir" -create

    This will prompt for a new password for the wallet.

2.  Create the entry for the database user name and password that are currently
    hardcoded in your Python scripts.  Use either of the methods shown below.
    They will prompt for the wallet password that was set in the first step.

    **Method 1 - Using an Easy Connect string**::

        mkstore -wrl "/home/oracle/wallet_dir" -createCredential dbhost.example.com/orclpdb1 myuser myuserpw

    **Method 2 - Using a connect name identifier**::

        mkstore -wrl "/home/oracle/wallet_dir" -createCredential mynetalias myuser myuserpw

    The alias key ``mynetalias`` immediately following the
    ``-createCredential`` option will be the connect name to be used in Python
    scripts.  If your application connects with multiple different database
    users, you could create a wallet entry with different connect names for
    each.

    You can see the newly created credential with::

        mkstore -wrl "/home/oracle/wallet_dir" -listCredential

3.  Skip this step if the wallet was created using an Easy Connect String.
    Otherwise, add an entry in :ref:`tnsnames.ora <optnetfiles>` for the connect
    name as follows::

        mynetalias =
            (DESCRIPTION =
                (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost.example.com)(PORT = 1521))
                (CONNECT_DATA =
                    (SERVER = DEDICATED)
                    (SERVICE_NAME = orclpdb1)
                )
            )

    The file uses the description for your existing database and sets the
    connect name alias to ``mynetalias``, which is the identifier used when
    adding the wallet entry.

4.  Add the following wallet location entry in the :ref:`sqlnet.ora
    <optnetfiles>` file, using the ``DIRECTORY`` you created the wallet in::

        WALLET_LOCATION =
            (SOURCE =
                (METHOD = FILE)
                (METHOD_DATA =
                    (DIRECTORY = /home/oracle/wallet_dir)
                )
            )
        SQLNET.WALLET_OVERRIDE = TRUE

    Examine the Oracle documentation for full settings and values.

5.  Ensure the configuration files are in a default location or set TNS_ADMIN is
    set to the directory containing them.  See :ref:`optnetfiles`.

With an Oracle wallet configured, and readable by you, your scripts
can connect using::

    connection = cx_Oracle.connect(dsn="mynetalias", encoding="UTF-8")

or::

    pool = cx_Oracle.SessionPool(externalauth=True, homogeneous=False, dsn="mynetalias",
                         encoding="UTF-8")
    pool.acquire()

The ``dsn`` must match the one used in the wallet.

After connecting, the query::

    SELECT SYS_CONTEXT('USERENV', 'SESSION_USER') FROM DUAL;

will show::

    MYUSER

.. note::

    Wallets are also used to configure TLS connections.  If you are using a
    wallet like this, you may need a database username and password in
    :meth:`cx_Oracle.connect()` and :meth:`cx_Oracle.SessionPool()` calls.

**External Authentication and Proxy Authentication**

The following examples show external wallet authentication combined with
:ref:`proxy authentication <proxyauth>`.  These examples use the wallet
configuration from above, with the addition of a grant to another user::

    ALTER USER mysessionuser GRANT CONNECT THROUGH myuser;

After connection, you can check who the session user is with:

.. code-block:: sql

    SELECT SYS_CONTEXT('USERENV', 'PROXY_USER'),
           SYS_CONTEXT('USERENV', 'SESSION_USER')
    FROM DUAL;

Standalone connection example:

.. code-block:: python

    # External Authentication with proxy
    connection = cx_Oracle.connect(user="[mysessionuser]", dsn="mynetalias", encoding="UTF-8")
    # PROXY_USER:   MYUSER
    # SESSION_USER: MYSESSIONUSER

Pooled connection example:

.. code-block:: python

    # External Authentication with proxy
    pool = cx_Oracle.SessionPool(externalauth=True, homogeneous=False, dsn="mynetalias",
                         encoding="UTF-8")
    pool.acquire(user="[mysessionuser]")
    # PROXY_USER:   MYUSER
    # SESSION_USER: MYSESSIONUSER

The following usage is not supported:

.. code-block:: python

    pool = cx_Oracle.SessionPool("[mysessionuser]", externalauth=True, homogeneous=False,
                         dsn="mynetalias", encoding="UTF-8")
    pool.acquire()


Operating System Authentication
-------------------------------

With Operating System authentication, Oracle allows user authentication to be
performed by the operating system.  The following steps give an overview of how
to implement OS Authentication on Linux.

1.  Login to your computer. The commands used in these steps assume the
    operating system user name is "oracle".

2.  Login to SQL*Plus as the SYSTEM user and verify the value for the
    ``OS_AUTHENT_PREFIX`` parameter::

        SQL> SHOW PARAMETER os_authent_prefix

        NAME                                 TYPE        VALUE
        ------------------------------------ ----------- ------------------------------
        os_authent_prefix                    string      ops$

3.  Create an Oracle database user using the ``os_authent_prefix`` determined in
    step 2, and the operating system user name:

   .. code-block:: sql

        CREATE USER ops$oracle IDENTIFIED EXTERNALLY;
        GRANT CONNECT, RESOURCE TO ops$oracle;

In Python, connect using the following code::

       connection = cx_Oracle.connect(dsn="mynetalias", encoding="UTF-8")

Your session user will be ``OPS$ORACLE``.

If your database is not on the same computer as python, you can perform testing
by setting the database configuration parameter ``remote_os_authent=true``.
Beware this is insecure.

See `Oracle Database Security Guide
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-37BECE32-58D5-43BF-A098-97936D66968F>`__ for more information about
Operating System Authentication.

Privileged Connections
======================

The ``mode`` parameter of the function :meth:`cx_Oracle.connect()` specifies
the database privilege that you want to associate with the user.

The example below shows how to connect to Oracle Database as SYSDBA:

.. code-block:: python

    connection = cx_Oracle.connect("sys", syspwd, "dbhost.example.com/orclpdb1",
            mode=cx_Oracle.SYSDBA, encoding="UTF-8")

    cursor = con.cursor()
    sql = "GRANT SYSOPER TO hr"
    cursor.execute(sql)

This is equivalent to executing the following in SQL*Plus:

.. code-block:: sql

    CONNECT sys/syspwd AS SYSDBA

    GRANT SYSOPER TO hr;


Starting and Stopping Oracle Database
=====================================

cx_Oracle has the capability of starting up the database using a privileged
connection. This example shows a script that could be run as the 'oracle'
operating system user who administers a local database installation on Linux.
It assumes that the environment variable ``ORACLE_SID`` has been set to the SID
of the database that should be started:

.. code-block:: python

    # the connection must be in PRELIM_AUTH mode to perform startup
    connection = cx_Oracle.connect("/",
            mode = cx_Oracle.SYSDBA | cx_Oracle.PRELIM_AUTH)
    connection.startup()

    # the following statements must be issued in normal SYSDBA mode
    connection = cx_Oracle.connect("/", mode = cx_Oracle.SYSDBA, encoding="UTF-8")
    cursor = connection.cursor()
    cursor.execute("alter database mount")
    cursor.execute("alter database open")

Similarly, cx_Oracle has the ability to shutdown the database using a
privileged connection. This example also assumes that the environment variable
``ORACLE_SID`` has been set:

.. code-block:: python

    # need to connect as SYSDBA or SYSOPER
    connection = cx_Oracle.connect("/", mode = cx_Oracle.SYSDBA)

    # first shutdown() call must specify the mode, if DBSHUTDOWN_ABORT is used,
    # there is no need for any of the other steps
    connection.shutdown(mode = cx_Oracle.DBSHUTDOWN_IMMEDIATE)

    # now close and dismount the database
    cursor = connection.cursor()
    cursor.execute("alter database close normal")
    cursor.execute("alter database dismount")

    # perform the final shutdown call
    connection.shutdown(mode = cx_Oracle.DBSHUTDOWN_FINAL)


.. _netencrypt:

Securely Encrypting Network Traffic to Oracle Database
======================================================

You can encrypt data transferred between the Oracle Database and the Oracle
client libraries used by cx_Oracle so that unauthorized parties are not able to
view plain text values as the data passes over the network.  The easiest
configuration is Oracle’s native network encryption.  The standard SSL protocol
can also be used if you have a PKI, but setup is necessarily more involved.

With native network encryption, the client and database server negotiate a key
using Diffie-Hellman key exchange.  This provides protection against
man-in-the-middle attacks.

Native network encryption can be configured by editing Oracle Net’s optional
:ref:`sqlnet.ora <optnetfiles>` configuration file, on either the database
server and/or on each cx_Oracle 'client' machine.  Parameters control whether
data integrity checking and encryption is required or just allowed, and which
algorithms the client and server should consider for use.

As an example, to ensure all connections to the database are checked for
integrity and are also encrypted, create or edit the Oracle Database
``$ORACLE_HOME/network/admin/sqlnet.ora`` file.  Set the checksum negotiation
to always validate a checksum and set the checksum type to your desired value.
The network encryption settings can similarly be set.  For example, to use the
SHA512 checksum and AES256 encryption use::

    SQLNET.CRYPTO_CHECKSUM_SERVER = required
    SQLNET.CRYPTO_CHECKSUM_TYPES_SERVER = (SHA512)
    SQLNET.ENCRYPTION_SERVER = required
    SQLNET.ENCRYPTION_TYPES_SERVER = (AES256)

If you definitely know that the database server enforces integrity and
encryption, then you do not need to configure cx_Oracle separately.  However
you can also, or alternatively, do so depending on your business needs.  Create
a ``sqlnet.ora`` on your client machine and locate it with other
:ref:`optnetfiles`::

    SQLNET.CRYPTO_CHECKSUM_CLIENT = required
    SQLNET.CRYPTO_CHECKSUM_TYPES_CLIENT = (SHA512)
    SQLNET.ENCRYPTION_CLIENT = required
    SQLNET.ENCRYPTION_TYPES_CLIENT = (AES256)

The client and server sides can negotiate the protocols used if the settings
indicate more than one value is accepted.

Note that these are example settings only. You must review your security
requirements and read the documentation for your Oracle version. In particular
review the available algorithms for security and performance.

The ``NETWORK_SERVICE_BANNER`` column of the database view
`V$SESSION_CONNECT_INFO
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-9F0DCAEA-A67E-4183-89E7-B1555DC591CE>`__ can be used to verify the
encryption status of a connection.

For more information on Oracle Data Network Encryption and Integrity,
configuring SSL network encryption and Transparent Data Encryption of
data-at-rest in the database, see `Oracle Database Security Guide
<https://www.oracle.com/pls/topic/lookup?ctx=dblatest&
id=GUID-41040F53-D7A6-48FA-A92A-0C23118BC8A0>`__.


Resetting Passwords
===================

After connecting, passwords can be changed by calling
:meth:`Connection.changepassword()`:

.. code-block:: python

    # Get the passwords from somewhere, such as prompting the user
    oldpwd = getpass.getpass("Old Password for %s: " % username)
    newpwd = getpass.getpass("New Password for %s: " % username)

    connection.changepassword(oldpwd, newpwd)

When a password has expired and you cannot connect directly, you can connect
and change the password in one operation by using the ``newpassword`` parameter
of the function :meth:`cx_Oracle.connect()` constructor:

.. code-block:: python

    # Get the passwords from somewhere, such as prompting the user
    oldpwd = getpass.getpass("Old Password for %s: " % username)
    newpwd = getpass.getpass("New Password for %s: " % username)

    connection = cx_Oracle.connect(username, oldpwd, "dbhost.example.com/orclpdb1",
            newpassword=newpwd, encoding="UTF-8")

.. _connsharding:

Connecting to Sharded Databases
===============================

The :meth:`cx_Oracle.connect()` and :meth:`SessionPool.acquire()`
functions accept ``shardingkey`` and ``supershardingkey`` parameters
that are a sequence of values used to identify the database shard to
connect to.  Currently only strings are supported for the key values.
See `Oracle Sharding
<https://www.oracle.com/database/technologies/
high-availability/sharding.html>`__ for more information.
