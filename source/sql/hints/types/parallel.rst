.. _sql-hints-types-parallel:
 
Parallel Execution Hints
------------------------
Not all SQL statements can be run in parallel.
All DML statements, including subqueries, can be run in parallel, which means that multiple blocks can be selected, inserted, deleted, or updated simultaneously.
For parallelized DDL statements, multiple blocks are being created/altered and written in parallel.
The DDL statements that can be run in parallel are:
 
* ``CREATE INDEX``
* ``CREATE TABLE ... AS SELECT``
* ``ALTER INDEX ... REBUILD``
* ``ALTER INDEX ... [ REBUILD | SPLIT ] PARTITION``
* ``ALTER TABLE ... MOVE``
* ``ALTER TABLE ... [ MOVE | SPLIT | COALESCE ] PARTITION``
 
Note that for the CTAS statement it is possible to perform a parallel DML (i.e. ``SELECT``) operation but write the data to disk serially, which means that it is not a parallel DDL operation.
We do not intend to dwell on such technicalities though.
 
The parallel execution of DDL statements requires that it be enabled either at the level of the session or by specifying the appropriate ``PARALLEL`` clause for the statement.
When set manually for a table or index with the ``ALTER { TABLE | INDEX } obj_name PARALLEL dop`` , the degree of parallelism (DOP) used to be for both subsequent DDL *and* DML statements *prior to 12c*.
Beware of that trap!
Nevertheless, since this is a section on optimizer hints, we have no intention of delving into the specifics on non-hinted parallel execution.
 
As of 11gR2 Oracle has had the ``PARALLEL( dop )`` and ``NO_PARALLEL ( tab_name_or_alias )`` for individual statements rather than sessions or objects.
The degree of parallelism ``dop`` is optional, and if omitted Oracle computes it for you; the minimum degree of parallelism is 2.
The ``PARALLEL`` hint will cause all access paths than can use parallelism to use it; in essence, the hint authorizes the optimizer to use parallelism.
The hint can be supplied to the ``SELECT``, ``INSERT``, ``UPDATE``, ``DELETE``, or ``MERGE`` bits of a statement.
 
Instead of supplying ``dop``, you can also write a) ``DEFAULT``, which means that the DOP is equal to the number of CPUs available on all instances multiplied by the value of the ``PARALLEL_THREADS_PER_CPU`` initialization parameter, b) ``AUTO``, which causes the optimizer to decide on the degree of parallelism or whether to run the statement serially, or c) ``MANUAL``, for which the degree of parallelism is determined by the objects in the statement.
 
The ``PARALLEL`` hint can also be set for specific objects in a SQL statement as follows: ``PARALLEL( tab_name_or_alias  dop )``.
You may also provide ``DEFAULT`` as an alternative to ``dop``; its behaviour is identical to the statement-level's hint.
 
In `Expert Oracle SQL`_ (p.152) it is noted that when inserting data in parallel *before* committing causes subsequent selects to fail *until* the data is committed.
The reason is that a `direct path write`_ can sometimes be used by parallel DML statements, especially inserts.
The rows for a direct path write are not in the :term:`SGA` and must be read from disk.
However, before the data is committed there is no fresh data to read from disk!
 
The ``NO_PARALLEL`` hint overrides the ``PARALLEL`` parameter supplied at the creation or alteration of any table.
 
In a similar fashion you can instruct Oracle to scan index ranges in parallel with the ``PARALLEL_INDEX( tab_name_or_alias  index_name  dop )``.
With ``NO_PARALLEL_INDEX( tab_name_or_alias  index_name )`` you can disable parallel index range scans.
In both hints, ``index_name`` is optional.
 
With ``PQ_CONCURRENT_UNION`` you force the optimizer to process ``UNION [ ALL ]`` operations in parallel.
``NO_PQ_CONCURRENT_UNION`` disables concurrent processing of said set operations.
 
When the distribution of the values of the join keys for a parallel join is highly skewed because many rows have the same join key values, parallelizing a join can be troublesome as the load is not easily distributed among the query servers.
To that end Oracle introduced the ``PQ_SKEW( tab_name_or_alias )`` hint, which informs the optimizer of data skew in the join keys.
Oracle requires `a histogram on the join expression`_ as otherwise it will probe rows at random to discover the skew; it also seems that only single inner joins are supported.
Similarly, there is a ``NO_PQ_SKEW( tab_name_or_alias )`` to advise the optimizer that most rows do not share the same join keys.
In both hints, ``tab_name_or_alias`` is the hash join's probe row source.
 
CTAS and ``INSERT INTO ... SELECT`` statements' distribution of rows between producers and consumers can be controlled with the ``PQ_DISTRIBUTE( tab_name_or_alias  distribution )`` hint.
The value of ``distribution`` can be one of the following:
 
* ``NONE``: no distribution, which is ideal when there is no skew, so the overhead of distributing rows can be avoided. It is important to be aware that each query server munches between 512 KB and 1.5 MB (with compression) of :term:`PGA` memory.
* ``PARTITION``: rows are distributed from producers to consumers based on ``tab_name_or_alias``'s partition information, which is best used when producer and consumer operations cannot be combined, there are more partitions than query servers, and there is no skew across partitions.
* ``RANDOM``: rows are distributed from the consumers to the consumers in a round-robin fashion, which is applicable when the data is skewed.
* ``RANDOM_LOCAL``: rows are distributed from the consumers to the consumers on the same RAC node in a round-robin fashion, which eliminates inter-node communication.
 
For joins it is also possible to use the hint in a slightly modified form: ``PQ_DISTRIBUTE( tab_name_or_alias  outer_distribution  inner_distribution )``.
`All possible values`_ are summarized in the table below.
 
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
|``outer_distribution`` | ``inner_distribution`` | Explanation                                                           | Use Case                                                                  |
+=======================+========================+=======================================================================+===========================================================================+
| ``HASH``              | ``HASH``               | Rows of both tables are mapped with a hash function on the join keys. | Tables have comparable sizes and join uses hash-join or sort-merge join.  |
|                       |                        | Each query server performs the join between pair of resulting         |                                                                           |
|                       |                        | partitions.                                                           |                                                                           |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
| ``BROADCAST``         | ``NONE``               | Rows of *outer* table are broadcast to each query server; rows of     | *Outer* table is small compared to inner table: inner-table size          |
|                       |                        | inner table are partitioned randomly.                                 | multiplied by number of query servers must be greater than outer-table    |
|                       |                        |                                                                       | size.                                                                     |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
| ``NONE``              | ``BROADCAST``          | Rows of *inner* table are broadcast to each query server; rows of     | *Inner* table is small compared to outer table: inner-table size          |
|                       |                        | outer table are partitioned randomly.                                 | multiplied by number of query servers must be less than outer-table size. |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
| ``PARTITION``         | ``NONE``               | Rows of *outer* table are mapped using partitioning of inner table;   | Number of partitions of *outer* table is roughly equal to number of query |
|                       |                        | inner table must be partitioned on join keys.                         | servers.                                                                  |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
| ``NONE``              | ``PARTITION``          | Rows of *inner* table are mapped using partitioning of outer table;   | Number of partitions of *outer* table is roughly equal to number of query |
|                       |                        | outer table must be partitioned on join keys.                         | servers.                                                                  |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
| ``NONE``              | ``NONE``               | Each query server joins a pair of matching partitions.                | Tables are equipartitioned on join keys.                                  |
|                       |                        |                                                                       |                                                                           |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
 
Please note that the last entry corresponds to the full partition-wise join we talked about :ref:`earlier <sql-join-partitions>`.

Finally, we have ``PQ_FILTER``, which tells Oracle how to process rows for correlated subqueries.
The following table shows all four parameter values, how the rows on the left-hand side and right-hand side of the filter are processed, and when best to use a particular parameter.

+------------+-------------------------------+----------+-----------------------------------------------------------------------------------------------+
| Parameter  | LHS                           | RHS      | Use Case                                                                                      |
+============+===============================+==========+===============================================================================================+
| ``HASH``   | Parallel: hash distribution   | Serial   | No skew in LHS data distribution                                                              |
+------------+-------------------------------+----------+-----------------------------------------------------------------------------------------------+
| ``NONE``   | Parallel                      | Parallel | No skew in LHS data distribution *and* LHS distribution best avoided (e.g. many rows in LHS ) |
+------------+-------------------------------+----------+-----------------------------------------------------------------------------------------------+
| ``RANDOM`` | Parallel: random distribution | Serial   | Skew in LHS data distribution                                                                 |
+------------+-------------------------------+----------+-----------------------------------------------------------------------------------------------+
| ``SERIAL`` | Serial                        | Serial   | Overhead of parallelization too high (e.g. few rows in LHS)                                   |
+------------+-------------------------------+----------+-----------------------------------------------------------------------------------------------+

Query Transformation Hints
--------------------------
Again, all hints in this category, save for the generic ``NO_QUERY_TRANSFORMATION`` hint, come in couples:

* ``FACT`` / ``NO_FACT``
* ``MERGE`` / ``NO_MERGE``
* ``NO_EXPAND`` / ``USE_CONCAT``
* ``REWRITE`` / ``NO_REWRITE``
* ``UNNEST`` / ``NO_UNNEST``
* ``STAR_TRANSFORMATION`` / ``NO_STAR_TRANSFORMATION``

With ``NO_QUERY_TRANSFORMATION`` you disable all query transformation that the optimizer can perform.
What the hint does *not* disable, though, are transformations that the optimizer *always* applies, such as the count transformation, predicate move-around, filter push-down, distinct elimination, and select-list pruning.
This is of course no hint for a production environment, and it should only be used for testing purposes.

Generic Transformations
^^^^^^^^^^^^^^^^^^^^^^^
We have already briefly seen the ``NO_MERGE( view_name_or_alias )`` hint.
It prohibits the optimizer from merging views in a query.
Similarly, you can force Oracle to merge (inline) views with ``MERGE( view_name )``.

.. note::
   ``MERGE`` and ``NO_MERGE`` have nothing to do with the sort-merge join!

When the view contains a ``GROUP BY`` clause or ``DISTINCT`` operator (or ``UNIQUE``) operator, the ``MERGE`` hint only works if `complex view merging`_ is enabled.
The delayed evaluation of these operations can either improve or worsen performance, so use these hints wisely and sparingly.
For instance, join conditions may reduce the data volume to be grouped or sorted, which may be beneficial to performance.
Likewise, it can be advantageous to aggregate data as early as possible to deal with less data in subsequent operations.
The optimizer uses the cost to determine whether it is better to merge views or not.
Complex view merging also allows uncorrelated ``IN``-subqueries to be merged into the main SQL statement.

``USE_CONCAT`` always enables the ``OR``-expansion, which transforms combined ``OR``-conditions or ``IN``-lists in the ``WHERE`` clause into a compound query with the ``UNION ALL`` operator.
Whether the cost with such an ``OR``-expansion is truly lower than without it is irrelevant: when specified Oracle does as instructed.
``NO_EXPAND`` makes the optimizer discard the ``OR``-expansion as a possible query transformation.

Subquery unnesting can be forced without regard for the cost with the ``UNNEST`` hint.
It combines subqueries in the ``WHERE``, such as in ``IN``-lists, into the ``FROM`` clause, which opens the door to more access paths for the optimizer to tinker with.
Without subquery unnesting, Oracle treats the main query and its subqueries as separate statements: the subqueries are executed, and their results are used to run the main query.
Subquery unnesting is possible if and only if the resulting join statement is guaranteed to returns the same rows as the original statement, for instance thanks to a primary key, and the subquery does not contain any aggregate functions.
``NO_UNNEST`` is, as you may have guessed, used to disable subquery unnesting.
Oracle unnests subqueries automatically unless hinted, regardless of cost expected.

Materialized views that have been created with the ``ENABLE QUERY REWRITE`` clause can be used to provide data to queries that do not explicitly call these materialized view in their ``FROM`` clauses.
Contrary to regular views, which are nothing but stored queries, materialized views store the result sets of the queries that define them and regularly refresh the data.
Materialized views are particularly useful for queries that are run often, as a snapshot of the data is taken and stored, so the data does not have to be calculated from scratch every time a user asks for it.
However, some users may not be aware of these materialized views, which is why they are executing their own queries that ask for the same data as contained in the materialized views.
With ``REWRITE`` you allow people to benefit from the data of these materialized views; the hint has an optional parameter, which is the name of the materialized view.
Typically, Oracle does this automatically when it determines that such a `rewrite`_ is beneficial.
If successful, it shows up in the execution plan as ``MAT_VIEW REWRITE``.

``NO_REWRITE`` overrides the ``ENABLE QUERY REWRITE`` clause, if present.
This can be helpful if you know that the data in the materialized view is stale compared to the source tables, and your query needs the current state of affairs.

The Star Transformation
^^^^^^^^^^^^^^^^^^^^^^^
In many data warehouses and OLAP databases that power business intelligence solutions, the dimensional rather than the entity-relationship data model is the gold standard. [#datamodels]_
Fact tables contain all information pertinent to a user's queries, and they can easily be joined to so-called dimension tables with more details on the dimensions listed in the fact table.
The schema for such databases resembles what we refer to as a snowflake schema.
In such instances, a star transformation can be useful, to which end Oracle has introduced the ``STAR_TRANSFORMATION`` hint.
When specified, Oracle does not guarantee that it will be used.

A requirement for the star transformation is that there be a `single-column bitmap index on all foreign-key columns of the fact table`_ that participate in the join.
The star transformation progresses in two steps:

#. Transform the original query with the join into a query with the fact table in the ``FROM`` clause and the dimension tables as subqueries in the ``WHERE`` clause to filter rows from the fact table based on the dimensions' values or ranges.
   The bitmap indexes are then combined with the bitmap ``AND``-operation to select only the rows that satisfy all dimensional constraints.
   The advantage is that all the dimension tables are logically joined with the fact table only once rather than once for each dimension table.
   Which join method is used depends on what the optimizer decides is best, although typically for large data volumes a hash join is chosen.
#. Adjoin the rows from the dimension tables to the fact table using the best access method available to the optimizer, which is typically a full table scan because dimension tables are often relatively small.

We have said that Oracle does not always perform a star transformation, even though the ``STAR_TRANSFORMATION`` hint is specified.
This is even true when all prerequisites, such as said bitmap indexes on the fact table, are met.
In fact, the optimizer calculates the best plan without the transformation and only then compares it to the best plan with the transformation.
Based on the costs of both plans, it picks one, which may not always be the one with the transformation enabled.
One such case is when a large fraction of the rows in the fact table need to be fetched, for instance because the constraints on the dimension tables are not selective enough.
It is then often advantageous to do a full table scan with multi-block reads.

Most of the time, database developers are told that bind variables are the key to great performance.
When your query has bind variables, the star transformation will never be used though.

Another instance when star transformations are never applied is when *fact* tables are accessed remotely, that is through a database link.
Dimension tables may, however, be on different Oracle database instances.

Anti-joins, fact tables that are unmerged or partitioned views, and dimension tables that appear both in the ``FROM`` clause and as subqueries in the ``WHERE`` clause are a few other party poopers for the star transformation.

The ``FACT( tab_name_or_alias )`` hint can be used to inform the optimizer which table should be considered the fact table.
``NO_FACT`` is exactly the opposite.

.. _`complex view merging`: http://blogs.oracle.com/optimizer/entry/optimizer_transformations_view_merging_part_2
.. _`single-column bitmap index on all foreign-key columns of the fact table`: http://docs.oracle.com/cd/B19306_01/server.102/b14223/schemas.htm#CIHGHEFB
.. _`Expert Oracle SQL`: http://www.apress.com/9781430259770
.. _`direct path write`: http://www.toadworld.com/platforms/oracle/w/wiki/793.direct-path-write.aspx
.. _`a histogram on the join expression`: http://oracle-randolf.blogspot.de/2014/05/12c-hybrid-hash-distribution-with-skew.html
.. _`All possible values`: http://docs.oracle.com/database/121/SQLRF/sql_elements006.htm#BABCJHAF
.. _`rewrite`: http://docs.oracle.com/database/121/DWHSG/qrbasic.htm#DWHSG018

.. rubric:: Notes

.. [#datamodels] We have no intention of starting a debate on the data model paradigms of Kimball and Inmon. The interested reader will find plenty of insightful articles `on the internet <http://searchbusinessintelligence.techtarget.in/tip/Inmon-vs-Kimball-Which-approach-is-suitable-for-your-data-warehouse>`_.