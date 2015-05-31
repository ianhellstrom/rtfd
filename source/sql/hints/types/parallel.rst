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

.. _`Expert Oracle SQL`: http://www.apress.com/9781430259770
.. _`direct path write`: http://www.toadworld.com/platforms/oracle/w/wiki/793.direct-path-write.aspx
.. _`a histogram on the join expression`: http://oracle-randolf.blogspot.de/2014/05/12c-hybrid-hash-distribution-with-skew.html
.. _`All possible values`: http://docs.oracle.com/database/121/SQLRF/sql_elements006.htm#BABCJHAF
