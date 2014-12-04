.. _sql-subqueries:

**********
Subqueries
**********
`Subqueries`_ come in different shapes and forms: scalar subqueries, nested (single- or multi-row) subqueries (in the ``WHERE`` clause), correlated subqueries, inline views (in the ``FROM`` clause), and factored subqueries (in the ``WITH`` clause), which are also known as common table expressions (CTEs) outside Oracleland.
They are pretty versatile constructs as they can appear in almost any clause of DML statements with the exception of the ``GROUP BY`` clause.
 
Many of the subquery types are interchangeable.
Inline subqueries can be rewritten as factored subqueries, and factored subqueries can often be rewritten as inline subqueries; recursive factored subqueries are the exception to this rule.
Recursive factored subqueries can, nevertheless, typically be written as hierarchical queries using the ``CONNECT BY ... START WITH ...`` syntax.
Similarly, you can write a correlated subquery as a join, but not every join can become a correlated subquery.
 
Scalar Subqueries
=================
Let's start with the conceptually easiest type of the subqueries: scalar subqueries.
Usually a scalar subquery in the ``SELECT`` needs to be evaluated for each row of the outer query (i.e. the part without the scalar subquery).
If the scalar subquery calculates, say, a sum of a certain values from a large table, this means that the sum has to be calculated many times.
Since it makes no sense to scan the same table over and over again for a value that really only needs to be computed once, Oracle Database 12c has the ability to unnest the scalar subquery, which means that it can convert the scalar subquery into, for instance, a ``GROUP BY`` view that is (outer-)joined to the table without the scalar subquery.
 
`Tanel Poder`_ has written about this feature, and his advice on the matter is spot-on too: you rarely need scalar subqueries in the ``SELECT`` clause and rewriting your scalar subqueries as joins opens up a whole collection of optimization options that Oracle can tinker with in the background.
Oracle has had the ability to unnest scalar subqueries prior to 12c, but there was still a `bug associated with scalar subquery unnesting`_ until 11.2.
 
Nested and Correlated Subqueries
================================
Nested and correlated subqueries show up in the ``WHERE``  clause of a SQL statement.
Whereas a scalar subquery returns one row and one column, a single-row subquery returns one row but multiple columns, and a multi-row subquery returns multiple rows and multiple columns.
Whenever the subquery does not reference columns from the outer query, we speak of a nested subquery, otherwise it is called a correlated subquery.
 
For multi-row nested subqueries it is important to note that the ``ANY``, ``ALL``, and ``SOME`` operators can sometimes be equivalent to ``IN``-lists, which is why they do not often show up in production code even though Oracle loves them at certification exams.
For instance, ``WHERE col_name = ANY ( ... )`` is equivalent to ``WHERE col_name IN ( ... )``, and ``WHERE col_name <> ALL ( ... )`` is exactly the same as ``WHERE col_name NOT IN ( ... )``, where the ellipsis indicates any valid, nested multi-row subquery.
In fact, `Oracle already does some of these transformations`_ (and more) automatically.
 
Indexes are primarily used in the filters of the ``WHERE`` clause, as we have discussed before.
This includes predicates with nested or correlated subqueries too.
As such it is often advantageous to rewrite ``NOT EXISTS`` (anti-join) as ``EXISTS`` (semi-join) because it allows Oracle to use an index.
 
Related to the topic of semi-joins is whether there is any difference among the following three options that are commonly found in code:
 
* ``WHERE EXISTS (SELECT * FROM tab_name ...)``,
* ``WHERE EXISTS (SELECT 1 FROM tab_name ...)``,
* ``WHERE EXISTS (SELECT col_name FROM tab_name ...)``.
 
The short answer: no.
 
The long answer is that the cardinality estimates may be slightly different, but in general the optimizer still chooses the same execution plan, for these differences are rarely the cause for the optimizer to think differently.
The cost *is* affected by the cardinality estimates, but these are likely to be close together if the statistics for ``tab_name`` are representative.
Oracle stops processing as soon as it obtains the first matching entry from ``tab_name``.
This question basically boils down to the age-old question whether ``COUNT(*)`` is the same as ``COUNT(1)``, and the answer is `affirmative`_.
 
Subquery Unnesting
------------------
The Oracle query optimizer has basically two types of transformations at its disposal:
 
#. Cost-based transformations, which are performed only when the optimizer believes it will improve the performance;
#. Heuristic transformations, which are generally applied regardless of what the optimizer believes.
 
Some heuristic transformations are really no-brainers because they cannot impact the performance negatively:
 
* ``COUNT`` conversion: whenever a ``COUNT(*)`` is equivalent to a ``COUNT(col_not_null)`` Oracle can use a ``BITMAP CONVERSION COUNT`` to speed up the SQL statement.
* ``DISTINCT`` is eliminated whenever Oracle knows the operation is redundant, for instance when all columns of a primary key are involved.
* Columns in a ``SELECT``-list of a subquery are removed whenever they are not referenced, which is known as select-list pruning.
* Filters can be pushed down into subqueries when appropriate, so that the amount of data is minimized as soon as possible.
* Predicates can also be moved around from one subquery to another whenever an inner or natural join of two (or more) query blocks is performed where only one of the blocks has a particular predicate, but transitive closure guarantees that is also applies to the other subqueries.
  Aggregations and analytic functions are known to throw a spanner in the works for filter push-downs and predicate move-arounds.
 
Subquery unnesting is an interesting beast in the sense that is always applied irrespective of the impact on the performance.
As such, it should be classified as a heuristic transformation were it not for the fact that it can be disabled with a hint.
 
`Oracle automatically unnests subqueries`_ (in the ``WHERE`` clause) when possible and `merges the body of the subquery into the body of the statement`_ or `rewrites the nested subquery as an inline view`_, which of course opens up new optimization avenues, unless the nested subquery:
 
* contains the ``ROWNUM`` pseudocolumn;
* contains a set operator;
* contains an aggregate function or ``GROUP BY`` clause;
* contains a correlated reference to a query block that is not the immediate outer query block of the subquery;
* is a hierarchical subquery.
 
Why is unnesting important?
Without it, Oracle needs to evaluate a (correlated) subquery *for each row of the outer table*.
With it, it has a whole array of access paths and join options it can go through to improve the execution.
 
When you execute ``SELECT * FROM dual WHERE 0 NOT IN (NULL)`` you will receive no rows, as expected.
After all, null may be 0, it may not be.
Before Oracle Database 11g, a column that could be null would prevent Oracle from unnesting the subquery.
With the null-aware anti-join in 11g and above this is no longer the case, and Oracle can unnest such subqueries.
 
As of Oracle Database 12c there is the so-called null-accepting semi-join, which extends the semi-join algorithm, indicated by ``SEMI NA`` in the execution plan.
This is relevant for correlated subqueries that have a related ``IS NULL`` predicate, like so: ``WHERE col_name IS NULL OR EXISTS ( SELECT 1 FROM ... )``
The null-accepting semi-join checks for null columns in the join column of the table on the left-hand side of the join.
If it is null, the corresponding row is returned, otherwise a semi-join is performed.
 
So, you may be wondering, 'If Oracle already unnests correlated subqueries, is there any reason to use correlated subqueries instead of joins?'

A correlated subquery is perfectly acceptable when your outer query already filters heavily and the correlated subquery is used to find corresponding matches.
This often happens when you do a simple lookup, typically in a PL/SQL (table) function in an API.
 
Beware of nulls in the subquery of anti-joins though: whenever one or more rows return a null, you won't see any results.
A predicate such as ``col_name NOT IN (NULL, ...)`` always evaluates to null.
Analogously, it is important that you inform Oracle of nulls, or the absence thereof, in case you decide to explicitly rewrite a nested or correlated subquery as a join, as it may assist Oracle in determining a better execution plan.
Remember: the more information the optimizer has, the better its decisions.
 
Combined Nested Subqueries
--------------------------
Sometimes you need to filter for two different (aggregated) values from a subquery.
Basically, you have `two options`_.
 
Option 1:
 
.. code-block:: sql
   :linenos:
 
    SELECT
      ...
    FROM
      tab_name
    WHERE
      col_name = ( SELECT ... FROM sub_tab_name ... )
    AND another_col_name = ( SELECT ... FROM sub_tab_name ... );
 
Option 2:

.. code-block:: sql
   :linenos:
 
    SELECT
      ...
    FROM
      tab_name
    WHERE
      ( col_name, another_col_name ) =
      (
        SELECT aggregation(...), another_aggregation(...) FROM sub_tab_name ...
      );
 
The second option is to be preferred because the number of lookups in ``sub_tab_name`` is minimized: ``col_name`` and ``another_col_name`` are retrieved in the same round trip, potentially for each relevant row of ``tab_name``.
 
Subqueries with ``DISTINCT``
----------------------------
Let's take a look at two queries:
 
.. code-block:: sql
   :linenos:
 
    SELECT
      DISTINCT
      some_fancy_function(col_name) AS col_alias
    FROM
      tab_name;
 
.. code-block:: sql
   :linenos:
     
    SELECT
      some_fancy_function(col_name) AS col_alias
    FROM
      (
        SELECT DISTINCT col_name FROM tab_name
      );
 
Which one will run faster?
 
Well, in the first case, a full-table scan is done to fetch the columns, after which the function ``some_function`` is applied to each column, and finally Oracle looks for distinct values.
In the second case, Oracle scans the table ``tab_name``, returns only distinct values for ``col_name``, and then applies the function to the results of the inline view.
The function is invoked for every row of ``tab_name`` in the former query, whereas in the latter it is only called for every distinct ``col_name``.
Therefore, the bottom query will have better runtime performance.
 
Important to note is that the result sets of both may not be the same though.
Suppose ``col_name`` contains the following *distinct* entries: 0, 1, and 2.
Let's also assume that the function we want to apply is ``SIN(col_name*c_pi)``, where ``c_pi`` can be a global (PL/SQL) constant defined in a package as ``ACOS(-1)``.
In case you have already forgotten geometric functions from basic calculus -- shame on you! --: the sine function is zero at all multiples of :math:`{\pi}`.
The former query will therefore return only one row with ``col_alias`` equal to zero, whereas the latter will return three rows, all zero.
 
Functions that lead to the same result set in both cases are known as bijective maps in mathematical circles.
They map distinct input (domain) to distinct output (range); there is a one-to-one correspondence between the domain and the range of the function.
A non-mathematical example that shows similar behaviour as our sine function is ``SUBSTR(col_name, 1, 1)``.
It takes the first character of each ``col_name``, which means that ``'Jack'`` and ``'Jill'`` are both mapped to ``'J'``.
 
So, when you *know* that the function you apply is a bijection, then you can rewrite your original query in the format that typically runs faster.
 
Sometimes you can even avoid a ``DISTINCT`` (with the associated costly sort operation) in a main query's ``SELECT``-list altogether by opting for a semi-join (i.e. ``EXISTS``) instead.
This is common when you want unique entries from the main table but only when there is a match in another table for which there are multiple rows for one original row, that is, there is a one-to-many relationship from the main to the other (subquery) table.
 
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
 
.. _Subqueries: http://docs.oracle.com/database/121/SQLRF/queries007.htm#SQLRF52357
.. _Tanel Poder: http://blog.tanelpoder.com/2013/08/13/oracle-12c-scalar-subquery-unnesting-transformation
.. _bug associated with scalar subquery unnesting: http://timurakhmadeev.wordpress.com/2011/06/28/scalar-subquery-unnesting
.. _Oracle already does some of these transformations: http://oracle-base.com/articles/misc/all-any-some-comparison-conditions-in-sql.php
.. _affirmative: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:40208915257337
.. _Oracle automatically unnests subqueries: http://blogs.oracle.com/optimizer/entry/optimizer_transformations_subquery_unesting_part_2
.. _merges the body of the subquery into the body of the statement: http://docs.oracle.com/database/121/SQLRF/queries008.htm#SQLRF52358
.. _rewrites the nested subquery as an inline view: http://blogs.oracle.com/optimizer/entry/optimizer_transformations_subquery_unesting_part_1
.. _two options: http://www.akadia.com/services/ora_query_tuning.html
.. _Tony Hasler in Oracle Expert SQL: http://www.apress.com/9781430259770
.. _global temporary tables: http://www.dba-oracle.com/t_tuning_sql_subqueries.htm
.. _use the TEMP rather than the UNDO tablespace: http://www.dba-oracle.com/t_with_clause.htm
.. _Pro Oracle SQL: http://www.apress.com/9781430232285
.. _Databaseline: http://wp.me/p4zRKC-2G
