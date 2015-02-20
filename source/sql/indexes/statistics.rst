.. _sql-indexes-stats:
 
Statistics
==========
Contrary to what some people may have heard, a *balanced* search tree is, as the name suggests, *always* — read that again, please — always balanced.
It is a myth that you have to rebuild the index whenever the performance of your queries is below par.
There are extremely rare cases when `Oracle recommends`_ that you rebuild the indexes but in almost all cases you do not have to rebuild your indexes. [#rebuild]_


Oracle nowadays automatically collects statistics, so once you create an index, Oracle takes care of the rest.
You can see the schedule and some basic information about the statistics collection with the following statement:
 
.. code-block:: sql
   :linenos:
 
    SELECT
      *
    FROM
      dba_autotask_client
    WHERE
      client_name = 'auto optimizer stats collection'
    ;
 
For most databases the automatic statistics collection is sufficient.
If, however, your database has tables that are being deleted and truncated between collection runs, then it can make sense to go `gather optimizer statistics manually`_.
 
When you create an index, Oracle automatically gathers optimizer statistics because it needs to do a full scan anyway.
As of Oracle Database 12c, the same piggybacking is done for the statistics collection of create-table-as-select (CTAS) and insert-as-select (IAS) operations, which is quite nifty; histograms require additional data scans, so these are not automatically gathered.
The execution plans of CTAS and IAS statements show whether statistics are being collected at runtime: ``OPTIMIZER STATISTICS GATHERING``, right below the ``LOAD AS SELECT`` operation.
 
If you change the definition of an index, you may want to update the statistics.
Please coordinate with the DBA to avoid unwanted side effects, such as degrading the performance of all but your own queries because invalidation of execution plans; gathering statistics does not lock the table, it's like running multiple queries against it. [#invaplan]_

.. _`Oracle recommends`: http://blogs.oracle.com/sysdba/entry/when_to_rebuild_index
.. _`gather optimizer statistics manually`: http://docs.oracle.com/database/121/TGSQL/tgsql_stats.htm#TGSQL415

.. rubric:: Notes

.. [#rebuild] The index clustering factor indicates the correlation between the index order and the table order; the optimizer takes the clustering factor into account for the ``TABLE ACCESS BY INDEX ROWID`` operation. A high ratio of leaf nodes marked for deletion to leaf nodes (> 0.20), a low value of percentage used (< 0.80), and a clustering factor (see ``DBA_INDEXES``) close to the number of rows (instead of the number of blocks) in the table (see ``DBA_SEGMENTS``) are indicators that your indexes may benefit from rebuilding. If the clustering index is close to the number of rows, then the rows are ordered randomly.

.. [#invaplan] The ``DBMS_STATS.AUTO_INVALIDATE`` option can be used to ensure that Oracle does not invalidate all cursors immediately, which can cause a significant CPU spike. Instead, Oracle uses a rolling cursor invalidation based on internal heuristics.
