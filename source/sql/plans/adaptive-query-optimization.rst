.. _sql-adaptive:

Adaptive Query Optimization
===========================
An important new feature in Oracle Database 12c is `adaptive query optimization`_, which consists of two components: adaptive plans and adaptive statistics.
The optimizer's statistics collector has the ability to detect whether its cardinality estimates are different from the actual number of rows seen by the individual operations in the plan.
If the difference is significant, then the plan or a at least a portion of it can be modified on the fly to avoid suboptimal performance of the initial execution of a SQL statement.
Plans are also automatically re-optimized, which means that *after* the initial execution of a SQL statement and *before* the next execution of the same statement, Oracle checks whether its estimates were off, and if so determines an alternative plan based on corrected, stored statistics.

Statistics feedback allows re-optimization based on erroneous cardinality estimates discovered during the execution.
Tables without statistics, queries with multiple filter predicates on a table, and queries with predicates that include complex operators are candidates for statistics feedback; if multiple filters on columns that are correlated are issued, then the combined data can become skewed as compared to the original data, which is something the optimizer is not aware of before the execution of the statement.

You can see whether a statement can be re-optimized by querying ``v$sql``: the column ``is_reoptimizable`` holds the information you seek.
The next time you execute the statement a new plan will be generated, for which the flag ``is_reoptimizable`` will be ``N``.
Similarly, joins can be adapted at runtime; only nested loops can be swapped for hash joins and vice versa.

Oracle Database 12c also introduced another distribution method for the parallel execution of queries: ``HYBRID HASH``.
This allows the optimizer to defer its distribution method until execution at which point more up-to-date information on the number of rows involved can be used.
Immediately in front of the ``PX SEND HYBRID HASH`` operation there is the work of the ``STATISTICS COLLECTOR``.
If the statistics collector discovers that the actual number of rows buffered is less the threshold of 2·DOP, the distribution method will go from ``HASH`` to ``BROADCAST``.
If it finds out that the actual number of rows buffered is more than the threshold, the distribution method will always be ``HASH``.
Re-optimization of parallelizable SQL statements is done with the performance feedback mechanism, which allows the optimizer to improve the degree of parallelism for repeated SQL statements; AutoDOP has to be enabled though.

Something you can see in the notes section of execution plans is whether a SQL plan directive was used.
A SQL plan directive is automatically created when automatic re-optimization of query expressions takes place.
It instructs the optimizer what to do when a certain query expression is encountered, for example to employ ``DYNAMIC_SAMPLING`` because of cardinality misestimates.
Information about the SQL plan directive can be obtained from ``dba_sql_plan_directives`` and ``dba_sql_plan_dir_objects``.
When a certain expression is encountered by the optimizer, the SQL plan directive type listed (e.g. ``DYNAMIC_SAMPLING``) is employed.

As in previous versions, ``EXPLAIN PLAN`` only returns the execution plan preferred by the optimizer. 
The function ``DBMS_XPLAN.DISPLAY_CURSOR`` shows the plan actually used by the query executor.
To see all information, please use the following statement:

.. code-block:: sql
   :linenos:
   
   SELECT 
     * 
   FROM 
     table ( DBMS_XPLAN.DISPLAY_CURSOR ( FORMAT => '+adaptive' ) )
   ;

.. note::
   When dynamic statistics is enabled (``ALTER SESSION SET OPTMIZER_DYNAMIC_SAMPLING = 11)``), the time it takes to parse a statement will go up.
   More information can be found on the Oracle's `web page for the query optimizer`_.

.. _`adaptive query optimization`: http://www.oracle.com/technetwork/database/bi-datawarehousing/twp-optimizer-with-oracledb-12c-1963236.pdf
.. _`web page for the query optimizer`: http://www.oracle.com/technetwork/database/bi-datawarehousing/dbbi-tech-info-optmztn-092214.html