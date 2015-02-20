.. _sql-subqueries-inline_views:
 
Inline Views and Factored Subqueries
====================================
We tend to agree wholeheartedly with `Tony Hasler in Oracle Expert SQL`_ (pp. 9-16) when it comes to the question whether to prefer inline views to factored subqueries or the other way round.
Some organizations have rules that instruct developers to use factored subqueries only when they are re-used in the same statement.
When a subquery is read multiple times, such as in a recursive common table expression, factored subqueries can improve the performance of your SQL statements, especially with the materialize or cache hint.
There are, however, no performance problems associated with factored subqueries when they are queried only once, so it's more a matter of style than performance in these cases.
Whenever it is more advantageous to (temporarily) materialize the factored subquery, Oracle will automatically do so.
Of course, this does not always work, especially when statistics are unavailable or not representative of the current situation.
 
Interestingly, recursive factored subqueries can sometimes perform better than traditional solutions, especially for hierarchical queries.
A detailed example is provided by Ian Hellström on `Databaseline`_ for the multiplication across the branches of a hierarchy, where an approach with a recursive factored subquery is shown to outperform the standard Oracle solution with ``CONNECT BY`` by several orders of magnitude.
 
Before the advent of factored subqueries, developers were often told that `global temporary tables`_ were the cure for bad subquery performance.
That is no longer the case because either Oracle already materializes the factored subquery or you can force Oracle do to so with ``/*+ materialize */``.
Similarly, you can provide the hint ``/*+ CACHE */``, so that Oracle caches the factored subquery, which can improve performance when the SQL statement accesses the factored subquery more than once.
As of Oracle Database 12c, there is a session variable ``temp_undo_enabled`` that allows you to `use the TEMP rather than the UNDO tablespace`_ for temporary tables, materializations, and factored subqueries.
 
The only reason you may *not always* want to use factored subqueries is that in certain DML statements only inline views are permitted.
Factored subqueries are easier to read and debug, hands down, and the performance is often superior too.
So, unless you have a compelling reason, for instance syntax restrictions or performance, although the latter is rarely the case, stay away from inline views and go for glory with factored subqueries.
For recursive subqueries and subqueries that need to be accessed multiple times in the same SQL statement, factored subqueries are pretty much your only option.
 
What is important, though, is -- and this is by no means restricted to inline views vs factored subqueries -- that you give your subquery factors meaningful names: ``q1``, ``a``, or ``xx``   do *not* qualify as meaningful names.
 
There is one instance, and one instance only, where an ``ORDER BY`` clause in an inline view or factored subquery is acceptable: top-*N* queries or pagination.
If you only want the top-*N* rows based on some ordering, you simply need to sort the data.
In all other cases, an intermediate sort does not make sense and may negatively affect the runtime performance.
If the query blocks that work with the data from the sorted subquery perform several joins or sorts of their own, the effort of the initial sort is gone, including the time it took.
When data needs to be sorted you do that in the outer query (for an inline view) or final ``SELECT`` statement (for a factored subquery).
Often such unnecessary ``ORDER BY`` clauses are remnants of the development phase, so please make sure that you clean up your code afterwards.
 
Don't believe it?
Take a look at the following SQL statement:
 
.. code-block:: sql
   :linenos:
 
   WITH
      raw_data AS
      (
        SELECT ROWNUM AS rn FROM dual CONNECT BY ROWNUM <= 1000 --ORDER BY rn DESC
      )
    SELECT * FROM raw_data ORDER BY rn;
   
The execution plan for the statement with the ``ORDER BY`` in the subquery factor reads (on 12c):

.. code-block:: none
   :emphasize-lines: 7
 
    -----------------------------------------------------------------------------------------
    | Id  | Operation                        | Name | Rows  | Bytes | Cost (%CPU)| Time     |
    -----------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT                 |      |     1 |    13 |     4  (50)| 00:00:01 |
    |   1 |  SORT ORDER BY                   |      |     1 |    13 |     4  (50)| 00:00:01 |
    |   2 |   VIEW                           |      |     1 |    13 |     3  (34)| 00:00:01 |
    |   3 |    SORT ORDER BY                 |      |     1 |       |     3  (34)| 00:00:01 |
    |   4 |     COUNT                        |      |       |       |            |          |
    |   5 |      CONNECT BY WITHOUT FILTERING|      |       |       |            |          |
    |   6 |       FAST DUAL                  |      |     1 |       |     2   (0)| 00:00:01 |
    -----------------------------------------------------------------------------------------
 
For the statement *without* the unnecessary sorting it is (again on 12c):

.. code-block:: none
 
    ----------------------------------------------------------------------------------------
    | Id  | Operation                       | Name | Rows  | Bytes | Cost (%CPU)| Time     |
    ----------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT                |      |     1 |    13 |     3  (34)| 00:00:01 |
    |   1 |  SORT ORDER BY                  |      |     1 |    13 |     3  (34)| 00:00:01 |
    |   2 |   VIEW                          |      |     1 |    13 |     2   (0)| 00:00:01 |
    |   3 |    COUNT                        |      |       |       |            |          |
    |   4 |     CONNECT BY WITHOUT FILTERING|      |       |       |            |          |
    |   5 |      FAST DUAL                  |      |     1 |       |     2   (0)| 00:00:01 |
    ----------------------------------------------------------------------------------------
 
The difference is the additional ``SORT ORDER BY`` operation with Id = 3.
 
Oracle does have a so-called ``ORDER BY`` elimination that removes unnecessary sorting operations, such as in subqueries.
Such an elimination typically occurs when Oracle detects post-sorting joins or aggregations that would mess up the order anyway.
Important to note is that said elimination procedure does *not* apply to factored subqueries, which is why the ``SORT ORDER BY`` operation shows up in the execution plan above!
 
You can have fun with the order-by-elimination by enabling/disabling it with the hints ``ELIMINATE_OBY``/``NO_ELIMINATE_OBY``.
Again, please observe that this fiddling around with these hints only applies to inline views.
Similarly, you can use the ``NO_QUERY_TRANSFORMATION`` hint to disable *all* query transformations, as described by the authors in `Pro Oracle SQL`_ (pp. 45-46).
 
.. _`Tony Hasler in Oracle Expert SQL`: http://www.apress.com/9781430259770
.. _`global temporary tables`: http://www.dba-oracle.com/t_tuning_sql_subqueries.htm
.. _`use the TEMP rather than the UNDO tablespace`: http://www.dba-oracle.com/t_with_clause.htm
.. _`Pro Oracle SQL`: http://www.apress.com/9781430232285
.. _`Databaseline`: http://wp.me/p4zRKC-2G