.. _sql-subqueries-unnesting:
 
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
Let's also assume that the function we want to apply is ``SIN(col_name*gc_pi)``, where ``gc_pi`` can be a global (PL/SQL) constant defined in a package as ``ACOS(-1)``.
In case you have already forgotten geometric functions from basic calculus: the sine function is zero at all multiples of :math:`\pi`.
The former query will therefore return only one row with ``col_alias`` equal to zero, whereas the latter will return three rows, all zero.
 
Functions that lead to the same result set in both cases are known as bijective maps in mathematical circles.
They map distinct input (domain) to distinct output (range); there is a one-to-one correspondence between the domain and the range of the function.
A non-mathematical example that shows similar behaviour as our sine function is ``SUBSTR(col_name, 1, 1)``.
It takes the first character of each ``col_name``, which means that ``'Jack'`` and ``'Jill'`` are both mapped to ``'J'``.
 
So, when you *know* that the function you apply is a bijection, then you can rewrite your original query in the format that typically runs faster.
 
Sometimes you can even avoid a ``DISTINCT`` (with the associated costly sort operation) in a main query's ``SELECT``-list altogether by opting for a semi-join (i.e. ``EXISTS``) instead.
This is common when you want unique entries from the main table but only when there is a match in another table for which there are multiple rows for one original row, that is, there is a one-to-many relationship from the main to the other (subquery) table.

.. _`Oracle automatically unnests subqueries`: http://blogs.oracle.com/optimizer/entry/optimizer_transformations_subquery_unesting_part_2
.. _`merges the body of the subquery into the body of the statement`: http://docs.oracle.com/database/121/SQLRF/queries008.htm#SQLRF52358
.. _`rewrites the nested subquery as an inline view`: http://blogs.oracle.com/optimizer/entry/optimizer_transformations_subquery_unesting_part_1
.. _`two options`: http://www.akadia.com/services/ora_query_tuning.html
