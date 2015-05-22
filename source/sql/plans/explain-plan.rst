.. _sql-plans-explain-plan:

Explain Plan
============
As a developer, the execution plan is probably the best tool at your disposal to discover why a query performs the way it does *and*, if necessary, figure out what you can do about its performance.
A 2011 Oracle white paper called *Explain the Explain Plan*, which is available on the `OTN`_, and the documentation on `optimizer access paths`_ form the basis for most of what is written in this section.
Additionally, Jonathan Lewis has written an excellent series on the execution plan, the parts of which you can find `here`__, `here`__, and `here`__.

.. __: http://allthingsoracle.com/execution-plans-part-1-finding-plans
.. __: http://allthingsoracle.com/execution-plans-part-2-things-to-see
.. __: http://allthingsoracle.com/execution-plans-part-3-the-rule

So, how do you obtain the execution plan?

If you happen to work with `Oracle SQL Developer`_ you can simply press F10 to see the execution plan of the query selected.
It does not get much easier than that.
Similarly, you can execute ``EXPLAIN PLAN FOR your_query;`` and then run

.. code-block:: sql
   :linenos:
   
    SELECT
      *
    FROM
      table
        (
          DBMS_XPLAN.DISPLAY
            (
               'plan_table'  -- default plan table name
              , NULL         -- NULL to show last statement inserted into plan table
              , 'all'        -- show all available metrics
            )
        );

What you end up with is a tabular representation of the execution plan; the table is a top-down, left-to-right traversal of the execution tree.
Each operation is printed on a single line, together with more detailed information about that operation.
The sequence and types of operations depend on what the query optimizer considers to be the best plan, that is the one with the lowest cost.
The components that are factored into the cost of a query are

* Cardinality
* Access methods
* Join methods
* Join types
* Join orders
* Partition pruning
* Parallel execution

We shall now discuss each of these components separately.

.. _`Oracle SQL Developer`: http://www.oracle.com/technetwork/developer-tools/sql-developer
.. _`OTN`: http://www.oracle.com/technetwork/database/focus-areas/bi-datawarehousing/twp-explain-the-explain-plan-052011-393674.pdf
.. _`optimizer access paths`: http://docs.oracle.com/cd/E16655_01/server.121/e15858/tgsql_optop.htm

Cardinality
-----------
No, cardinality has nothing to do with the clergy.
What cardinality refers to is the uniqueness (or lack thereof) of data in a particular column of a database table.
It is a measure of the number of distinct elements in a column.
A low cardinality implies that a column has very few distinct values, and is thus not very selective.

In the context of execution plans, the cardinality shows the number of rows estimated to come out of each operation.
The cardinality is computed from table and column statistics, if available. [#autostats]_
If not, Oracle has to estimate the cardinality.
For instance, suppose you have an equality predicate in a single-table query, for which there is no histogram (i.e. no statistics).
The query optimizer will then assume a uniform distribution, so that the cardinality of each operation in the execution plan is calculated by dividing the total number of rows by the number of distinct values after the equality predicate has been applied.
The number of rounded up and shown in the column ``CARDINALITY``.

What can (negatively) impact the accuracy of the estimate and therefore the quality of the execution plan are i) data skew, ii) multiple single-column predicates on a single table, iii) function-wrapped columns in  ``WHERE`` clause predicates, and iv) complex expressions.

Interestingly, you can see runtime cardinality information by using the ``GATHER_PLAN_STATISTICS`` hint in your query, after which you have to execute ``SELECT * FROM table(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT=>'ALLSTATS LAST'))``.
The result shows you the estimated number of rows (``E-Rows``) and the actual number of rows (``A-Rows``), which can of course be quite different because of data skew.
Don't use this hint in a production environment as each query incurs some overhead.

Access Methods
--------------
The way in which Oracle accesses data is aptly called an access method.
Oracle has a bunch of access methods in its arsenal:

* A **full table scan** reads all rows from a heap-organized table and filters out the ones that do not match the ``WHERE`` clause predicates.
  Each formatted block of data under the high water mark (HWM) is read only once and the blocks are read in sequence.
  A multi-block read can be performed to speed up the table scan: several adjacent blocks are combined into a single I/O call.
  The ``DB_FILE_MULTIBLOCK_READ_COUNT`` parameter in the init.ora file defines the number of blocks that can be lumped into one multi-block read.
  
  A full table scan is typically employed if 
  
  * no index exists;
  * the index cannot be used because the query predicate has a function applied to an indexed column (in a non-function-based index);
  * the optimizer decides against an index skip scan because the query predicate does not include the leading edge of a (B-tree) index;
  * ``SELECT COUNT(*)`` is issued and the index contains nulls;
  * the table statistics are stale and the table has grown considerably since the statistics were last computed;
  * the query is not selective, so that a large portion of the rows must be accessed;
  * the cost of a full table scan is the lowest because the table is small, in particular the number of formatted blocks under the high water mark is smaller than ``DB_FILE_MULTIBLOCK_READ_COUNT``;
  * the table has a high degree of parallelism, which makes the optimizer biased towards full table scans;
  * the query uses the ``FULL`` hint.
    
* In a **table access by ROWID**, Oracle looks up each selected row of a heap-organized table based on its ROWID, which specifies the data file, the data block within that file, and the location of the row within that block.
  The ROWID is obtained either from the ``WHERE`` clause predicate or through an index scan.
  If the execution plan shows a line ``TABLE ACCESS BY INDEX ROWID BATCHED`` it means that Oracle retrieves a bunch of ROWIDs from the index and then tries to access rows in block order to reduce the number of times each block needs to be accessed.

* The ``SAMPLE`` and ``SAMPLE_BLOCK`` clauses (with a sample percentage below 100%) cause a **sample table scan**, which fetches a random sample of data from a heap-organized table. 
  Note that block sampling is only possible during full table scans or index fast scans.

* For an **index unique scan** only one row of a (B-tree) index or index-organized table will be returned because of an equality predicate on a *unique* index or a primary key constraint; the database stops looking for more rows once it has found its match because there cannot be any more matches thanks to the ``UNIQUE``  constraint.
  Hence, *all* columns of a unique index must be referenced with equality operators.
  Please observe the index itself must be unique: creating an index on a column and subsequently adding a ``UNIQUE`` or ``PRIMARY KEY`` constraint to the table is not enough, as the index is not aware of uniqueness; you are liable to end up with an index range scan. 

* An **index range scan** scans values in order.
  By default, Oracle stores and scans indexes and index-organized tables in ascending order.
  Oracle accesses adjacent index entries and uses the ROWID values to retrieve the rows in ascending order.
  If multiple index entries happen to have the same keys, Oracle returns the entries in ascending order by ROWID.
  
  The database chooses an index range scan if the leading columns of a *non-unique* index are specified in the ``WHERE`` clause or if the leading columns of a *unique* index have ranges rather than single values specified.
  Oracle navigates from the root blocks to the branch blocks where it reads the maximum values of the leading edge in the index for each leaf block that the branch blocks refer to.
  Because the database has no way of knowing in advance how many hits there are, it must visit each relevant leaf block and read each one until it does not find any more matches.
  The advantage of the index unique scan is that Oracle does not have to visit the leaf blocks at all, because once it has found its match it is done.
  Not so with the index range scan.
  
  A common gripe with the index range scan is in combination with the table access by ROWID method, especially if the index range scan includes filter predicates instead of mere access predicates.
  Filter predicates in conjunction with index range scans and tables access by ROWID can cause scalability issues, as the entire leaf node chain has to be accessed, read, and filtered.
  As more and more data is inserted, the time to access, read, and filter the data increases too.
    
  There is also an **index range scan descending**, which is basically the same beast; the only difference is that the rows are returned in descending order.
  The reason Oracle scans the index in descending rather than ascending order is because either the query has an ``ORDER BY ... DESC`` clause or the predicates on a key with a value less than instead of equal to or greater than a given value are specified.
  Another (obvious) cause is the ``INDEX_DESC`` hint.

* If the entire index is read in order, then we are dealing with an **full index scan**.
  A full index scan does not read every block in the index though.
  Instead it reads the root block and goes down the left-hand side (ascending scan) or right-hand side (descending scan) of the branch blocks until it reaches a leaf block.
  From there it reads the index on a block-by-block basis.

  The full index scan is used when one of the following situations arises:
  
  * a predicate references a (non-leading) column in an index;
  * no predicate is specified but all columns in the table as well as query are in the index, and at least one indexed column is not null;
  * the query has an ``ORDER BY`` clause on indexed non-null columns;
  * the query has a ``GROUP BY`` clause where all aggregation columns are present in the index.

  The query optimizer estimates whether it is cheaper to scan the index (full index scan) or table itself (full table scan); for index-organized tables the table and index are of course one and the same.
  
* A **fast full index scan** reads index blocks as they exist on disk.
  The index (entries on the leaf blocks) rather than the table is read in multi-block I/O mode.
  
  Oracle looks at a this access path whenever a query only asks for attributes in the index.
  It's an alternative to a full table scan when the index contains all columns needed for the query, and at least one column in the index key has a ``NOT NULL`` constraint.
  
  Because the index is not read in order (because of the multi-block I/O), a sort operation cannot be eliminated as with the full index scan.

* For queries with predicates on all but the leading column of a composite index, an **index skip scan** is a possibility.
  The index skip scan is also an option if the leading column has few distinct values.
  
  The optimizer logically splits a composite index into smaller sub-indexes based on the number of distinct values in the leading column(s) of the index.
  The lower the number of distinct values, the fewer logical sub-indexes, the more efficient the scan is.
  Index blocks that do not meet the filter condition on the non-leading column are immediately skipped, hence the name. 
  An index skip scan can of course only be efficient if the non-leading columns are highly selective.
 
* An **index join scan** is performed if all data can be retrieved from a combination of multiple indexes, which are hash-joined on the ROWIDs.
  Because all data is available in the indexes, no table access is needed.

  An index join is often more expensive than simply scanning the most selective index and subsequently probing the table.
  It cannot be used to eliminate a sort operation.
  
* Whereas in traditional B-tree indexes each entry point refers to exactly one row, a bitmap index's entry points refer to multiple rows.
  That is, each index key stores pointers to multiple rows. 
  Each bit corresponds to a possible ROWID; if the bit is set, then the row with the corresponding ROWID contains the key's value.
  The bit position is converted to an actual ROWID by a mapping function.
  Internally, Oracle stores the bitmap index in a B-tree structure for quick searching.
  
  Bitmaps are frequently used in data warehousing (OLAP) environments, where ad hoc queries are commonplace.
  Typically, a bitmap index is recommended if the indexed columns have low cardinality and the indexed table is rarely modified or even read-only.
  Bitmap indexes are not appropriate for OLTP applications, though.  
  If, for instance, the indexed column of a single row is updated, the database locks the index key entry, which points to many rows.
  Consequently, all these rows are locked.
  
  Back to business.
  If the predicate on a bitmap-indexed column contains an equality operator, the query optimizer considers the **bitmap index single value** access path to look up a single key value.
  A single bitmap is scanned for all positions containing a value of ``1``.
  All matching values are converted into ROWIDs, which in turn are used to find the corresponding rows.

* A B-tree index can have an index range scan for ranges of values specified in the ``WHERE`` clause.
  Its counterpart for bitmap indexes is the **bitmap index range scan**.
  
* A **bitmap merge** is typically preferred by the optimizer when bitmaps generated from a bitmap index range scan are combined with an ``OR`` operation between two bitmaps.

* When a query accesses a table in an indexed cluster, the optimizer mulls over a **cluster scan**.
  Tables in `table clusters`_ have common columns and related data stored in the same blocks.
  The proximity of the physical location of these shared columns reduces disk I/O when joining clustered tables and accessing related data.
  Table clusters are appropriate for tables that are rarely modified and do not require full table scans as in the case of retrieval of a lot of rows. 
  
  An index cluster is, as you might expect, a (B-tree) index on a cluster; the cluster index associates the cluster key with the database block address of the block with the relevant data.
  In a cluster scan, Oracle scans the index to obtain the database block addresses of all rows required.
  Because the index cluster is a separate entity, reading or storing rows in a table cluster requires at least two I/Os: one (or more) to retrieve/store the key value from/in the index, and one to read/write the row from/to the cluster.

* If you create a table cluster with the ``HASHKEYS`` clause, you end up with a hash cluster.
  A hash cluster is similar to an indexed cluster, only the index key is replaced with a hash function.
  All rows with the same hash are stored in the same data block.
  Because no separate cluster index exists as in the case with an indexed cluster, I/O can be usually halved.
  
  A **hash scan** is used to locate rows based on a hash value of the key in a hash cluster.
  A disadvantage of hash clusters is the absence of range scans on cluster keys that are not in the index, in which case a full table scan must be performed.

.. _sql-join-methods:

Join Methods
------------
Join methods refer to the way in which two row sources are joined with each other.
The query optimizer designates one table the outer table and the other the inner table.
If more than two tables need to be joined, the optimizer analyses all permutations to determine the optimal join sequence.

The join method dictates to a large degree the cost of joins:

* A **hash join** is usually preferred for (equi)joining large data sets.
  The query optimizer takes the smaller of two data sources to build a (deterministic) hash table in memory.
  It then scans the larger table and performs the same hashing on the join columns.
  For the in-memory hash table, the database probes each value and if it matches the predicate's conditions returns the corresponding row.
  
  If the smaller data set fits in memory, there is only the cost of a single read pass over both data sets.
  If the hash table does not fit in memory, then Oracle partitions it.
  The join is then performed partition by partition, which uses a lot of sort area memory and causes a lot of I/O to the temporary tablespace.

* A **nested loop join** is typically used when small subsets of tables are joined or if there is an efficient way of accessing the inner table, for example with an index lookup.
  For every row selected from the outer table, the database scans *all* rows of the inner table.
  If there is an index on the inner table, then it can be used to access the inner data by ROWID.
  
  The database can read several rows from the outer row source in a batch, which is typically part of an `adaptive plan`_.

  It is not uncommon to see two nested loops in the execution plan (as of 11g) because Oracle batches multiple I/O requests and process these with a vector I/O, which means that a set of ROWIDs is sent to the requesting operating system in an array.
  What Oracle does with two nested loops is basically the following:
  
  #. Iterate through the inner nested loop to obtain the requested outer source rows.
  #. Cache the data in the `PGA`_.
  #. Find the matching ROWID from the inner loop's inner row source.
  #. Cache the data in the PGA.
  #. Organize the ROWIDs for more efficient access in the cache.
  #. Iterate through the outer loop to retrieve the data based on the cached ROWIDs; the result set of the inner loop becomes the outer row source of the outer nested loop. 

* A **sort-merge join** is done when join conditions are inequalities (i.e. not an equijoin).
  It is commonly chosen if there is an index on one of the tables that eliminates a sort operation.
  The sort operation on the first data set can be avoided if there is such an index.
  The second data set will always be sorted, regardless of any indexes.
  It generally performs better for large data sets than a nested loop join.
  
  A sort-merge join proceeds in two steps:
  
  1. Sort join: both tables are sorted on the join key.
  1. Merge join: sorted tables are merged.

  Oracle accesses rows in the PGA instead of the SGA with a sort-merge join, which reduces logical I/O because it avoids repeated latching blocks in the database buffer cache; a :term:`latch` protects shared data structures in the SGA from simultaneous access.
  
* Whenever a table has no join conditions to any of the other tables in the statement, Oracle picks a **Cartesian join**, which is nothing but a Cartesian product of the tables.
  Because the number of rows of a Cartesian join of two tables is the product of the number of rows of the tables involved, it is generally used only if the tables are small.
  This join method is, however, very rare in production environments.

Partial join evaluation is available from Oracle Database 12c onwards.
It allows joined rows that would otherwise have to be eliminated by a ``SORT UNIQUE`` operation to be removed during the execution of the join an inner join or semi-join instead.
This optimization affects ``DISTINCT``, ``MIN()``, ``MAX()``, ``SUM(DISTINCT)``, ``AVG(DISTINCT)``, ``COUNT(DISTINCT)`` , branches of ``UNION``, ``MINUS``, and ``INTERSECT`` operators, ``[ NOT ] EXISTS`` subqueries, and so on.
For instance, a ``HASH JOIN > SORT UNIQUE`` is replaced by a ``HASH JOIN SEMI > HASH UNIQUE`` combo.

Join Types
----------
Join types are easier to explain because they are determined by what you have typed.
There are four categories of join types: i) inner joins, ii) outer joins (left, right, and full), iii) semi-joins, and iv) anti-joins.
Even though semi-joins and anti-joins are syntactically subqueries, the optimizer beats them until they accept their fate as joins.

Semi-joins can occur when the statement contains an ``IN`` or ``EXISTS`` clause that is not contained in an ``OR`` branch.
Analogously, anti-joins are considered if the statement contains a ``NOT IN`` or ``NOT EXISTS`` clause that is not contained in an ``OR`` branch.
Moreover, an anti-join can be used the statement includes an outer join and has ``IS NULL`` applied to a join column.

Join Orders
-----------
So, how does Oracle determine the order of joins?
The short answer is cost.
Cardinality estimates and access paths largely determine the overall cost of joins:

* Whenever a particular join results in at most one row (e.g. because of ``UNIQUE`` or ``PRIMARY KEY`` constraints) it goes first.
* For outer joins, the row-preserving (outer) table comes after the other tables to ensure that all rows that do not satisfy the join condition can be added correctly.
* When Oracle converts a subquery into an anti- or semi-join, the subquery's table(s) come after tables in the outer (connected/correlated) query block.
  Hash anti-joins and semi-joins can sometimes override the ordering though.
* If `view merging`_ is impossible, then all tables in the view are joined before joining the view to the tables outside the view.

.. _sql-partition-pruning:

Partition Pruning
-----------------
Partition pruning, which is also known as partition elimination, is used for partitioned tables when not all partitions need to be accessed.

Pruning is visible in the execution plan as integers in the columns ``PSTART``, the number of the first partition, and ``PSTOP``, the number of the last partition to be accessed. [#partition]_
If you do not see a number (e.g. ``KEY``), don't worry!
It means that Oracle was not able to determine the partition numbers at parse time but thinks that dynamic pruning (i.e. during execution) is likely to occur.
This typically happens when there is an equality predicate on a partitioning key that contains a function, or if there is a join condition on a partitioning key column that, once joined with a partitioned table, is not expected to join with all partitions because of a filter predicate.

For hash-partitioned tables, there can only be pruning if the predicate on the partition key column is an equality or ``IN``-list predicate.
If the hash partition key involves more than one column, then all these columns must appear in the predicate for partition pruning to be able to occur.

Parallel Execution
------------------
Sometimes Oracle executes statements in parallel, which can significantly improve the runtime performance of a query.
The query coordinator (QC) initiates the parallel execution of a SQL statement.
The parallel server :term:`processes` (PSPs) are responsible for the actual work that is done in parallel.
The coordinator distributes the work to the PSPs, and may have to do some of the work that cannot be done in parallel.
A classic example is the ``SUM(...)`` operation: the PSPs calculate the subtotals but the coordinator has to add the subtotals from each PSP to obtain the final tally.

All lines in the execution plan above ``PX COORDINATOR`` are taken care of by the query coordinator.
Everything below is done by the PSPs.

A granule is the smallest unit of work a PSP can perform.
Each PSP works exclusively on its own granule, and when it is done with the granule, it is given another one until all granules have been processed.
The degree of parallelism (DOP) is typically much smaller than the total number of granules.

The most common granules are block-based.
For block-based granules, the ``PX BLOCK ITERATOR`` iterates over all generated block range granules.
However, it is sometimes advantageous to make use of pre-existing data structures, such as in the case of partitioned tables.
For partition-based granules, each PSP will do work for all data in a single partition, which means that the number of sub-partitions that need to be accessed is typically larger than or equal to the degree of parallelism.
If there is skew in the sizes of the (sub-)partitions, then the degree of parallelism is generally smaller than the number of (sub-)partitions.
Partition-based granules show up in the execution plan as ``PX PARTITION``.
Whether Oracle uses block-based or partition-based granules cannot be influenced.

PSPs work together in pairs as producers and consumers (of rows).
Producers are below the ``PX SEND`` operation in the execution plan.
The line ``PX RECEIVE`` identifies consumers who eventually send their results to the query coordinator: ``PX SEND QC``.
Similar information is shown in the column ``TQ`` (table queue).

Occasionally data needs to be redistributed, especially in joins, which is shown in the columns ``IN-OUT`` and ``PQ Distrib`` (parallel query distribution) of the execution plan.
Producers scan the data sources and apply ``WHERE`` clause predicates, after which they send the results to their partner consumers who complete the join.
``IN-OUT`` shows whether a parallel operation is followed by another parallel operation (``P->P``) or a serial operation (``P->S``).
On the line with ``PX SEND QC`` you always encounter ``P->S``, because the PSPs send their results to the QC, which is a serial process.
If you happen to see ``P->S`` below that line it means that there is a serialization point.
This may be due to not having set the parallel degree on one or more objects in the query.
The flag ``S->P`` often indicates that there is a `serial bottleneck`_ because parallel processes have to wait for the serial process to finish.

You may also encounter ``PCWP`` and ``PCWC``, or parallel combined with parent and parallel combined with child, respectively.
This means that a particular step is executed in parallel including the parent or child step.
An example is a parallel nested loop join, for which each PSP scans the outer (driving) row source but does the index lookups on the inner row source too.
If such an operation carries the label ``BUFFER(ED)``, it tells you that Oracle needs to temporarily hold data between parallel processes, because there is no (parent) PSP available that can accept the data.

.. note::
   The last operation before ``PX SEND QC`` always shows a buffered `blocking operation`_.

If the amount of data that needs to be buffered is larger than what can reasonably fit into memory, the data needs to written to the temporary tablespace, after which the PSPs have to re-read it.
You will thus incur significant penalties for having your queries executed in parallel.

There are several methods that can show up in the column ``PQ Distrib`` of the execution plan:
	
* ``HASH``: a hash function is applied to the join columns, after which a particular consumer receives the row source based on the hash value. This is by far the most common distribution method.
* ``BROADCAST``: the rows of the smaller of two data sources are sent to all consumers in order to guarantee that the individual server processes are able to complete the join.
* ``RANGE``: each PSP works on a specific data range because of parallel sort operations, so that the coordinator merely has to present the PSPs' results in the correct order instead of a sort on the entire result set.
* ``KEY``: only one side in a partial partition-wise join is redistributed.
* ``ROUND ROBIN``: rows are distributed to PSPs one at a time in a circular fashion. This is either the final redistribution operation before sending data to the requesting process, or one of the earlier steps when no redistribution constraints are needed.
* ``RANDOM``: rows are randomly assigned to PSPs.
* ``PARTITION``: partial partition-wise join, for which the first row source is distributed in accordance with the partitions of the second row source. By the way, a full partition-wise join, that is for equi-partitioned row sources, does not require redistribution.
		
On real application cluster (RAC) databases the ``LOCAL`` suffix indicates that rows are distributed to consumers on the same RAC node.

Watch out for ``PX COORDINATOR FORCED SERIAL``, which means that the plan may look like Oracle prefers a parallel execution of the SQL statement but it doesn't really; when push comes to shove, Oracle picks a serial execution.

What is important to understand is that each data flow operation (DFO) in the execution plan can have its own degree of parallelism.
The degree of parallelism for each DFO at execution time may be quite different from the one determined by the optimizer because of a `downgrade`_ in the degree of parallelism.
Common causes are that the amount of CPUs available at execution time is different from what the optimizer assumed would be available, or that Oracle is unable to use similar amounts of PGA memory for the SQL areas because only a specific amount is allocated per PSP.

Even if at runtime the degree of parallelism is as expected, the parallel execution can only be efficient if the work is distributed evenly across the PSPs.
During development, the view ``v$pq_tqstat`` can help you with identifying skew across table queues.
Data skew or unequal partitions are the usual suspects, depending on the ``PX ITERATOR`` chosen by the query optimizer.
An insidious reason for distribution skew is the ``HASH`` distribution method.
The hashes are sometimes not uniformly distributed, generally because of an outer join for which all nulls are hashed to the same value, which leads to some PSPs receiving a larger-than-average share of data.
As a consequence, the remainder of PSPs are idle most of the time.

.. _`table clusters`: http://docs.oracle.com/cd/E16655_01/server.121/e17633/tablecls.htm#CNCPT608
.. _`adaptive plan`: http://docs.oracle.com/cd/E16655_01/server.121/e15858/tgsql_optcncpt.htm#TGSQL221
.. _`PGA`: http://docs.oracle.com/cd/E16655_01/server.121/e15857/tune_pga.htm#TGDBA346
.. _`view merging`: http://blogs.oracle.com/optimizer/entry/optimizer_transformations_view_merging_part_1
.. _`serial bottleneck`: http://www.toadworld.com/platforms/oracle/w/wiki/384.oracle-parallel-sql-monitoring.aspx
.. _`blocking operation`: http://www.oracle.com/technetwork/articles/database-performance/geist-parallel-execution-1-1872400.html
.. _`downgrade`: http://www.oracle.com/technetwork/articles/database-performance/geist-parallel-execution-2-1872405.html

.. rubric:: Notes

.. [#autostats] By default, Oracle determines all columns that need histograms based on usage statistics and the presence of data skew. You can manually create histograms with the ``DBMS_STATS.GATHER_TABLE_STATS`` procedure.

.. [#partition] If a table has *n* partitions (range partition) and *m* sub-partitions per partition, then the numbering is from 1 to *n* · *m*. The absolute partition numbers are the actual physical segments.
