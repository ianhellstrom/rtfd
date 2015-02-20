.. sql-subqueries-nested:
 
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

.. _`Oracle already does some of these transformations`: http://oracle-base.com/articles/misc/all-any-some-comparison-conditions-in-sql.php
.. _`affirmative`: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:40208915257337
