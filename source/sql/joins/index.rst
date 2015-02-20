.. _sql-joins:

*****
Joins
*****
Probably the most used operation in a relational database is the infamous join.
Apart from the semi- and antijoin that are basically subqueries, which we have already seen, there are roughly two types of joins: the inner and the outer join.
The inner join encompasses the ``[ INNER ] JOIN ... ON ...`` and ``NATURAL JOIN`` syntax alternatives.
For the outer join we have ``LEFT [ OUTER ] JOIN ... ON ...``, ``RIGHT [ OUTER ] JOIN ... ON ...``, ``FULL [ OUTER ] JOIN ... ON ...``, but also some more exotic options that were introduced in 12c and are mainly interesting for people migrating from Microsoft SQL Server: ``CROSS APPLY ( ... )``, ``OUTER APPLY ( ... )``.
The former is a variation on the ``CROSS JOIN``, whereas the latter is a variation on the ``LEFT JOIN``. ``[ FULL ] JOIN ... USING ( ... )`` can be used as both an inner and a full join.
 
As of 12c there is also a left lateral join, which can be employed with the ``LATERAL ( ... )`` syntax.
A `lateral view`_ is "an inline view that contains correlation referring to other tables that precede it in the ``FROM`` clause".
The ``CROSS APPLY`` is the equivalent of an inner lateral join, and ``OUTER APPLY`` does the same for outer lateral joins.
A Cartesian product of two sets (i.e. tables) is achieved with the ``CROSS JOIN``.
 
Oracle still supports the traditional syntax whereby the tables to be joined are all in the ``FROM`` clause, separated by commas, and join conditions are specified in the ``WHERE`` clause, either with or without the ``(+)`` notation.
This syntax is generally not recommended any longer, as it has some limitations that the ANSI standard syntax does not have.
Beware that there have been some bugs and performance with the ANSI syntax, although their number has decreased dramatically in more recent Oracle Database versions.
For a discussion on this topic we refer to `this thread`_ and the links provided therein.
 
Internally Oracle translates these various joins into join methods to access the data.
The options Oracle has at its disposal are:
 
* nested loops;
* hash join;
* sort-merge join, which includes the Cartesian join as it is a simple version of the standard sort-merge join.
 
Joining can, for obvious reasons, be a tad heavy on the RAM.
What databases do to reduce memory usage is `pipelining`_.
It is effectively the same as what pipelined table function (in PL/SQL) do, but we should not get ahead of ourselves.
Pipelining in joins means that intermediate results are immediately pipelined (i.e. sent) to the next join operation, thus avoiding the need to store the intermediate result set, which would have increased memory usage.
OK, we can't help ourselves and jump the PL/SQL queue a bit: in a table function the result set is stored in a collection before it is returned.
In a *pipelined* table function we pipe rows, which means that we do not store the result set in a collection but return each row as soon as it is fetched.
In ETL situations, where lots of merges and transformations are typically done, pipelining can improve the performance significantly because the memory usage is reduced and rows can be loaded as soon as the database has them ready; there is no need to wait for all rows to be computed.
 
There are a couple of things developers can do to optimize the performance of SQL statements with joins and the main thing is pick the right one, syntactically: if you really only need an inner join, don't specify a full join 'just in case'.
The more data Oracle has to fetch, the more I/O there is, and by taking data you may not need it is possible that Oracle chooses a join method that is not perhaps the best for your business case.
Another tool in a developer's toolbox to boost Oracle's performance when it has to perform complex joins is understanding the various join methods and whether your situation may warrant a method that is not chosen -- or even considered -- by the optimizer.
In such cases hints are invaluable.
 
Similarly, it is important that developers understand the difference between single-column predicates in the ``ON`` clause and the same predicates in the ``WHERE`` clause.
Here's an example:
 
Query 1a:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 5,7-10
  
   SELECT
     *
   FROM
     departments dept
   INNER JOIN
     employees emp
   ON
     dept.department_id = emp.department_id
   WHERE
     emp.last_name LIKE 'X%'
   ;
 
Query 1b:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 5,7-9
  
   SELECT
     *
   FROM
     departments dept
   INNER JOIN
     employees emp
   ON
     dept.department_id = emp.department_id
   AND emp.last_name LIKE 'X%'
   ;
  
Query 2:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 5,7-10
  
   SELECT
     *
   FROM
     departments dept
   LEFT JOIN
     employees emp
   ON
     dept.department_od = emp.department_id
   WHERE
     emp.last_name LIKE 'X%'
   ;
 
Query 3:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 5,7-9
  
   SELECT
     *
   FROM
     departments dept
   LEFT JOIN
     employees emp
   ON
     dept.department_id = emp.department_id
   AND emp.last_name LIKE 'X%'
   ;
 
For inner joins the only difference is when the clauses are evaluated: the ``ON`` clause is used to join tables in the ``FROM`` clause and thus comes first -- remember the query processing order from :ref:`before <sql-basics-proc-order>`?
``WHERE``-clause predicates are logically applied afterwards.
Nevertheless, Oracle can typically use the ``WHERE`` clause already when performing the join, in particular to filter rows from the join of the driving row source.
 
But we have not explicitly specified the driving row source.
Is there thus a difference in the results from Queries 1a and 1b?
 
And the answer is… (cue drum roll): no!
 
Query 1a, on the one hand, looks at all departments, looks for employees in the departments, and finally removes any matching rows from both tables that do not have an employee with a last name that begins with an 'X'.
Query 1b, on the other hand, takes the departments and returns rows when it finds a department that has an employee with a surname starting with an 'X'.
Both queries do exactly the same, so for inner joins there is no *logical* difference.
Personally, we would prefer Query 1a's syntax to 1b's, because the ``WHERE`` clause is unambiguous: it filters rows from the join.
A single-column predicate in the ``ON`` clause of an inner join is murky at best, and should be avoided, because its intentions are not as clear as in the case of the ``WHERE`` clause.
  
For the outer joins, the difference is very real.
Query 2 looks at all departments and joins the employees table.
If a department happens to have no employees, the department in question is still listed.
However, because of the ``WHERE`` clause only rows (i.e. departments and employees) with the column ``last_name`` beginning with an 'X' are returned.
So, even if a department has plenty of employees but none of them has a last name that starts with an 'X', no row for that department will be returned because logically the ``WHERE`` clause is applied to the result set of the join.
 
If we place the predicate in the ``ON`` clause, as in Query 3, we make it part of the outer join clause and thus allow rows to be returned from the left table (``departments``) even if there is no match from the right table (``employees``).
The situation for ``last_name`` is the same as for ``department_id``: if a department has no employees *or* a department has no employees with a surname that starts with an 'X', the department still shows up but with ``NULL`` for every column of ``employees`` because there are no employees that match the join criterion.
 
Anyway, we have already talked about joins methods :ref:`before <sql-join-methods>`, but it may be beneficial to take another look at the various methods and when Oracle decides to pick one and not the others.

.. _`lateral view`: http://optimizermagic.blogspot.de/2007/12/outerjoins-in-oracle.html
.. _`this thread`: http://stackoverflow.com/a/7892125
.. _`pipelining`: http://sql-performance-explained.com


.. toctree::
   :hidden:

   Nested Loops <nested-loops>
   Hash Join <hash-join>
   Sort-Merge Join <sort-merge-join>
   Performance: ON vs WHERE <performance>