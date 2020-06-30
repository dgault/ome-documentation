Search and indexing configuration
=================================

How Indexing works
------------------

Indexing is not driven by the user, but happens automatically in the
background and can be controlled by a number of settings listed under
:ref:`search_configuration`. The indexer runs periodically as defined
by :property:`omero.search.cron` and parses the latest batch of new or
modified objects in the database.

Upon successful completion, the persistent count in the ``configuration``
table will be incremented.

::

    omero=# select value from configuration where name = 'PersistentEventLogLoader.v2.current_id';
     value
    -------
     30983
    (1 row)

.. note::

   Presence of more than one ``PersistentEventLogLoader.*`` value in your
   database indicates that you have run indexing with multiple versions of the
   server. This is fine. To allow a new server version to force an update,
   the configuration key may be changed. For example,
   ``PersistentEventLogLoader.current_id`` became
   ``PersistentEventLogLoader.v2.current_id`` in :commit:`a5cb64a`.

.. _search-failures:

Missing search results
----------------------

If you are having any difficult with search results not appearing
timely, first you should start by checking the health of the
Indexer-0 process:

-  Check the server's log directory for a file named ``Indexer-0.log`` and
   monitor its progress (e.g. using ``tail`` or similar). If messages of the
   format:

   ::

       INFO  [   ome.services.fulltext.FullTextIndexer ] (3-thread-2) INDEXED 2 objects in 1 batch(es) [2483 ms.]

   are periodically being appended to the log file, then your indexer process
   may be running behind. You can either wait for it to catch up, or try
   increasing the search batch size in order to speed processing. See the
   section on the :property:`omero.search.batch` setting for more information.

-  If there are no updates to the ``Indexer-0.log`` file even when new images,
   tags, files, etc. are added to the server, then it is possible that the
   Indexer process has become stuck. It is possible to force a restart of the
   indexer using the :ref:`icegrid_tools` like so:

   ::

       > omero admin ice
       Ice 3.6.3   Copyright (c) 2003-2016 ZeroC, Inc.
       >>> server list
       Blitz-0
       DropBox
       FileServer
       Indexer-0
       ...
       >>> server stop Indexer-0

   You do not need to manually re-start the Indexer, as IceGrid will handle the
   creation of a new Indexer process automatically.

In case neither of the above seems to be the case, then your indexer is running
normally and more likely your index has been corrupted. You will need to
:ref:`re-index <search-reindexing>` OMERO. Reasons why this might have
occurred include:

-  Missing search terms are part of a very large text file. If the indexer's
   maximum file size limit is reached, a file may not be indexed.
   See the section on the :property:`omero.search.max_file_size` setting for
   more information on increasing this limit.

-  A bug in Lucene prior to OMERO 5.0.1 caused some documents to be "sealed"
   in that old search terms would return the document, but newer terms would
   **not**.

.. _search-reindexing:

Re-indexing
-----------

Background re-indexing
^^^^^^^^^^^^^^^^^^^^^^

Under most circumstances, you should be able to re-index the database while
the server is still running. If you need to make any adjustments to the server
configuration or the process heap size, first shut the server down and make
these changes before restarting the server. Use the following steps to
initiate a re-indexing.

-  Disable the search indexer process and stop any currently running indexer
   processes:

   ::

       $ omero admin reindex --prepare

-  Remove the existing search Indexes by deleting the contents of the
   :file:`FullText` subdirectory of your :property:`omero.data.dir`:

   ::

       $ omero admin reindex --wipe

-  Reset the indexer's progress counter in the database:

   ::

       $ omero admin reindex --reset 0

-  Re-enable/restart the indexer process:

   ::

       $ omero admin reindex --finish

Depending on the size of your database, it may take the indexer some time to
finish re-indexing. During this time, your OMERO server will remain available
for use, however the search functionality will be degraded until the
re-indexing is finished. See :ref:`search-monitoring` for information on how
long this should take.

.. note::

   Once you wipe your full-text directory, searches will return fewer or no
   results until re-indexing is complete.

Off-line re-indexing
^^^^^^^^^^^^^^^^^^^^

It is also possible to re-index the database with the server off-line. First,
shutdown the OMERO server as normal and make any adjustments to the
configuration that need to be made. Clear the contents of the :file:`FullText`
directory and reset the indexing's progress counter as above::

  $ omero admin reindex --wipe
  $ omero admin reindex --reset 0

Then run the off-line re-indexing command::

   $ omero admin reindex --foreground

Re-indexing the database in off-line mode will use a 1 GB heap by default, but
this can be specified on the command-line with the ``--mem`` argument::

   $ omero admin reindex --foreground --mem=2g

Other search configuration properties from :ref:`search_configuration` can be
set for the processing by setting the :envvar:`JAVA_OPTS` environment
variable::

   $ JAVA_OPTS="-Domero.search.max_partition_size=100000" bin/omero admin reindex --foreground

Once foreground indexing is complete, re-enable the background indexer as
above::

    $ omero admin reindex --finish

.. _search-monitoring:

Monitoring re-indexing
^^^^^^^^^^^^^^^^^^^^^^

During re-indexing, it is possible to estimate the percent indexed using the
following SQL command::


    omero=> select 'At ' ||  current_timestamp(0) || ', Percent indexed: ' || trunc(((select count(*) from eventlog el, configuration c where el.id < cast(c.value as int) and (c.name like 'PersistentEventLogLoader%')) * 1.0) / (select count(*) from eventlog) * 100, 2) || '%';
                      ?column?
    ----------------------------------------------------
     At 2014-06-14 07:54:37+00, Percent indexed: 70.90%
    (1 row)

This value is also logged periodically both when re-indexing in the background
and the foreground and is available via JMX. See :ref:`jvm_metrics` for more information.

The overall re-indexing performance depends significantly on the memory
settings and the size of the repository to index. The following table provides
estimates of the process duration based on re-indexing of existing production
servers of various sizes:

.. list-table::
  :header-rows: 1

  - * Re-indexing type
    * Re-indexing duration
    * Binary repository size
    * Indexer memory settings

  - * Background [1]_
    * 8h
    * 19TB
    * ``-Xmx4800m``

  - * Off-line
    * 6h30
    * 16TB
    * ``--mem 2g``

.. [1] :ome-users:`[ome-users] Re-indexing OMERO's search database  <2015-February/005038.html>`

.. seealso::
  :doc:`/developers/Modules/Search`
    Section of the developer documentation describing how to perform search
    queries against the server.
