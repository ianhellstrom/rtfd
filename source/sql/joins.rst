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
 
For inner joins the only difference is when the clauses are evaluated: the ``ON`` clause is used to join tables in the ``FROM`` clause and thus comes first -- remember the query processing order from :ref:`before <sql-proc-order>`?
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
 
Nested Loops
============
Whenever you have correlated row sources for a left lateral join, Oracle uses nested loops to perform the join.
Nested loops can, however, be used for uncorrelated row sources too, although that often requires some hint trickery, but more on that later when hints are in our focus.
 
Nested loops work by fetching the result from the driving row source and querying the probe row source for each row from the driving row source.
It's basically a nested ``FOR`` loop, hence the name.
The driving row source is sometimes also referred to as the leading row source, which is reflected in the hint ``/*+ leading(...) */`` that can be used to specify the leading object.
 
Nested loops scale linearly: if the row sources double in size, it takes roughly twice as much time to do the join with nested loops, provided that there is a relevant index on the probe row source.
If there is no such index, we lose the linear scalability, because Oracle has to visit each row in the probe row source, which means that in that case nested loops scale quadratically.
Appropriate indexes can often be tricky when the probe source is an inline view; Oracle typically chooses the table, if there is any, as the probe row source.
A somewhat related problem is that the same blocks in the table being probed may be visited many times because different rows are looked at each time.
For small driving row sources the nested loop join is often the best option.
 
Since 11g Oracle can prefetch nested loops, which shows up in the execution plan as the join operation being a child of a table access operation.
This enables the database to first think about what it does to the ROWIDs it obtains from the nested loops.
For instance, if the ROWIDs are all consecutive but not in the buffer cache, the table access operation can benefit from a multi-block read.
 
Multiple nested loops operations can occasionally show up in the execution plan for just one join, which indicates that Oracle used the nested-loop batching optimization technique.
What this method does is transform a single join of two row sources into a join of the driving row source to one copy of the probe row source that is joined to a replica of itself on ROWID; since we now have three row sources, we need at least two nested loops.
The probe row source copy that is used to perform a self join on ROWID is used to filter rows, so it will have a corresponding ``TABLE ACCESS BY ... ROWID`` entry in the execution plan.
This cost-based optimization can often reduce I/O although the execution plan may not transparently display the benefits.
 
Whatever is specified in the ``WHERE`` clause that is exclusively applicable to the driving row source is used by Oracle to filter rows as soon as possible, even though semantically the filter comes after the ``JOIN`` clause.
 
Oracle always uses nested loops for left lateral joins.
What makes lateral joins useful is that predicates derived from columns in the driving row source (i.e. the row source specified *before* the ``LATERAL`` keyword) can be used in the probe row source (i.e. the inline view that follows ``LATERAL``).
 
Beware of the cardinality estimates when you use the ``gather_optimizer_statistics`` hint: for nested loops the estimated row count is *per iteration*, whereas the actual row count is for *all iterations*, as mentioned by Tony Hasler in `Expert Oracle SQL`_ (p. 266).
 
Hash Join
=========
In a hash join, Oracle hashes the join key of the 'driving' row source in memory, after which it runs through the 'probe' row source and applies the hash to obtain the matches.
We have placed the words 'driving' and 'probe' in quotes to indicate that the nomenclature is slightly different for hash joins though still applicable.
Because of the hashing it is clear that an index on the probe row source will not improve the performance of the hash join.
The only indexes that are beneficial in a hash join are indexes on predicates in the ``WHERE`` clause, but that is — as we have said — not specific to hash joins at all.
Moreover, when the probe row source is table, a hash join does not visit blocks multiple times, since the database goes through the probe row source once.
In fact, if Oracle decides to do a full table scan on the probe row source it may also decide to do multi-block reads to bump its retrieval efficiency.
 
It's not all peachy though.
Suppose that the probe row source is huge but only very few rows match the join clause and that we have no predicate or one that is barely selective.
With a hash join he database visits many blocks that contain no data we're interested in, because we cannot retrieve the rows through an index.
Whether the balance is tipped in favour of nested loops with an index lookup of the probe row source, if of course available, that perhaps visits blocks over and over again, or a hash join with a single scan of each block depends mainly on the selectivity of the join condition and the clustering factor of the index.
In these cases it is often advantageous to fix the execution plan with hints once you have discovered and argued that one consistently outperforms the other.
 
Hash joins scale linearly too, but there is one caveat — isn't there always?
The entire hash table has to fit in memory.
If it does not, the hash table will spill onto disk, which ruins the linear scalability to the ground.
As always, select only the columns you need as the size of the hash table may increase dramatically and thus ruin the performance benefit when Oracle runs out of memory.
 
Another gotcha with hash joins is that they can only be used with equality join conditions.

.. _sql-join-trees:

Join Orders and Join Trees
--------------------------
When you hash-join several row sources with an inner join, Oracle can in principle swap the order without affecting the result set.
Why would it want to do that?
Well, the optimizer may discover that one particular join order is better than all the others.
For an inner join of :math:`{n}` tables, there are :math:`{n!}` possible join orders.
For four tables, we have :math:`{4! = 4\cdot 3\cdot 2\cdot 1 = 24}` possibilities.
So the chances are that there is at least one that is significantly better and one that is the absolute worst.
 
Let's take four tables: T1, T2, T3, and T4.
A so-called `left-deep join tree`_ is obtained in the following way:
 
#. Place T1's hash cluster in a workarea.
#. *Join T1 and T2. Call the intermediate result set J12.*
#. Place J12's hash cluster in a workarea.
#. Drop T1's workarea.
#. *Join J12 and T3. Call the intermediate result set J123.*
#. Place J123's hash cluster in a workarea.
#. Drop J12's workarea.
#. *Join J123 and T4.*
#. Drop J123's workarea.
 
The italicized items are the actual logical steps in the join order.
The left row source is always the driving row source in our notation, and the right row source is always the probe row source.
We can also write this succinctly as ( ( T1 → T2 ) → T3 ) → T4.
 
For a right-deep join tree we have the following steps:
 
#. Place T4's hash cluster in a workarea.
#. Place T3's hash cluster in a workarea.
#. Place T2's hash cluster in a workarea.
#. *Join T2 and T1. Call the intermediate result set J21.*
#. Place J21's hash cluster in a workarea.
#. Drop T2's workarea.
#. *Join T3 and J21. Call the intermediate result set J321.*
#. Place J321's hash cluster in a workarea.
#. Drop T3's workarea.
#. *Join T4 and J321.*
#. Drop all remaining workareas.
 
We can write this as T4 → ( T3 → ( T2 → T1 ) ).
 
What is hopefully clear from these sequences is that a left-deep join tree requires two concurrent workareas, whereas a right-deep join tree has as many workareas as row row sources.
So, why on earth do we ever want a right-deep join tree?
 
Suppose for a second that T1 is enormous and the remaining tables are relatively small, which often happens in data warehouses.
Just think of T1 as being the fact table (e.g. sales) and T2, T3, and T4 dimension tables (e.g. products, customers, and suppliers).
In a left-deep join tree we would create a large workarea with T1, and potentially do a couple of Cartesian joins on the dimension tables as these often do not have join conditions with one another.
This would leave us with a monstrous hash cluster for T1 that will likely not fit in memory.
Moreover, the hash clusters of the Cartesian joins of the dimension tables may also easily be more than Oracle can handle.
The right-deep join tree places the smaller tables in workareas and finally scans the large table T1 instead.
In doing so, we have more workareas but they are all likely to fit in memory, thus allowing us to feel the wind of linear scalability in our hair as we speed through the joins.
 
Let's not get carried away now.
How do we obtain a right-deep from a left-deep join tree?
We can go from a left-deep join tree to a right-deep join tree in the following manner:
 
#. Swap T4: T4 → ( ( T1 → T2 ) → T3 ).
#. Swap T3: T4 → ( T3 → ( T1 → T2 ) ).
#. Swap T2: T4 → ( T3 → ( T2 → T1 ) ).
 
Notice that in the first swap we have obtained an intermediate result set as a probe row source.
 
The corresponding (simplified) execution plans would look something like the ones shown below.
In particular, for the left-deep join tree we have:
 
.. code-block:: none
 
    -------------------------------------
    | Id | Operation             | Name |
    -------------------------------------
    | 0  | SELECT STATEMENT      |      |
    | 1  |  HASH JOIN            |      |
    | 2  |   HASH JOIN           |      |
    | 3  |    HASH JOIN          |      |
    | 4  |     TABLE ACCESS FULL |  T1  |
    | 5  |     TABLE ACCESS FULL |  T2  |
    | 6  |    TABLE ACCESS FULL  |  T3  |
    | 7  |   TABLE ACCESS FULL   |  T4  |
 
And for the right-deep join tree we see:
 
.. code-block::none
 
    -------------------------------------
    | Id | Operation             | Name |
    -------------------------------------
    | 0  | SELECT STATEMENT      |      |
    | 1  |  HASH JOIN            |      |
    | 2  |   TABLE ACCESS FULL   |  T4  |
    | 3  |   HASH JOIN           |      |
    | 4  |    TABLE ACCESS FULL  |  T3  |
    | 5  |    HASH JOIN          |      |
    | 6  |     TABLE ACCESS FULL |  T2  |
    | 7  |     TABLE ACCESS FULL |  T1  |
 
These are of course not all of Oracle's options.
`Bushy joins`_ (yes, they are really called that) or zigzag join trees have some of the row sources swapped but not all as in the case of left-deep and right-deep join trees.
An example of such a zigzag tree would be the following: ( T1 → T2 ) → ( T3 → T4 ).
To be specific, we obtain that particular join order as indicated:
 
#. Join T1 and T2. Call the intermediate result set J12.
#. Join T3 and T4. Call the intermediate result set J34.
#. Join J12 and J34.
 
Interestingly, bushy joins are *never* considered by the optimizer.
Hence, if you believe a bushy join to be the best join order possible, you have to force Oracle with the ``leading`` hint.

.. _sql-join-partitions:

Partitioned Hash Joins
----------------------
For two tables that are equijoined and both partitioned identically, Oracle does a `full partition-wise join`_, which shows up as a ``PARTITION HASH`` parent operation to the ``HASH JOIN`` in the execution plan.
Similarly it can pop up in a parallel SQL statement as ``PX PARTITION HASH``.
Each parallel query server reads data from a particular partition of the first table and joins it with the appropriate rows from the corresponding partition of the second table.
Query servers have no need to communicate to one another, which is ideal.
The only downside is if there is at least one partition that is significantly larger than all the others, as this may affect the balancing of the load.
 
When only one table is partitioned, Oracle can go with a (parallel) `partial partition-wise join`_.
It (re)partitions the other table on the fly based on the partitioning scheme of the reference table.
Once the partitioning is out of the way, the database proceeds as it does with a full partition-wise join.
 
It is `generally recommended`_ to use hash instead of range partitioning for partition-wise joins to be effective, mainly because of possible data skew that leads to some partitions being larger than others.
Furthermore, `the number of partitions in relation to the DOP`_ is relevant to the performance.
Ideally, the number of partitions is a multiple of the number of query servers.
 
Both hash and sort-merge joins are possible for full partition-wise joins.
More details can be found in the excellent book `Expert Oracle SQL`_.
 
Sort-Merge Join
===============
The sort-merge or simply merge join is actually rarely used by Oracle.
It requires both row sources to be sorted by the join columns from the get-go.
For equality join conditions, the sort-merge join combines both row sources like a zipper: nicely in-sync.
When dealing with ranged-based join predicates, that is everything except ``<>``, Oracle sometimes has to jump a bit back and forth in the probe row source as it strolls through the driving row source, but it pretty much does what a nested loop does: for each row from the driving row source pick the matches from the probe row source.
 
A Cartesian join is basically a sort-merge join, and it shows up as ``MERGE JOIN CARTESIAN`` in the execution plan.
It is Oracle's fall-back plan: if there is no join predicate, then it has no alternative as every row in the driving row source matches each and every row in the probe row source.
What is slightly different for the Cartesian join is that no actual sorting takes place even though the execution plan informs us of a ``BUFFER SORT`` operation.
This operation merely buffers, it does *not* sort.
When the Cartesian product of two row sources is relatively small, the performance should not be too horrendous.
 
A sort-merge join may be performed when there is no index on the join columns, the selectivity of the join columns is low, or the clustering factor is high (i.e. near the number of rows rather than the number of blocks, so the rows are ordered randomly rather than stored in order), so that nested loops are not really an attractive alternative any longer.
Similarly, a sort-merge join may be done instead of a hash join when the hashed row sources are too big to fit in memory.
Note that a sort-merge join may spill onto disk too, although that typically is not as bad to performance as with a hash join.
 
When one row source is already sorted and Oracle decides to go ahead with a sort-merge join, the other row source will *always* be sorted even when it is already sorted.
 
The symmetry of the sort-merge join is unique.
In fact, the join order does not make a difference, not even to the performance.
 
Join Performance: ``ON`` vs ``WHERE``
=====================================
Now that we are equipped with a better appreciation and understanding of the intricacies of the various join methods, let's revisit the queries from the introduction.
 
Queries 1a and 1b are logically the same and Oracle will treat them that way.
First, let's assume there there is an index on ``department_id`` in both tables.
Such an index is only beneficial to nested loops because that particular column is in the join clause.
As such, the ``employees`` table is likely to become the driving row source, for a filter like ``LIKE last_name = 'X%'`` is probably very selective in many instances, which means that the number of iterations will be relatively low.
While accessing the ``employees`` table, Oracle will apply the filter because it knows that single-column join conditions in the ``ON`` clause of inner joins are the same as predicates in the ``WHERE`` clause.
The database will do so either with a lookup if the relevant index on ``employees`` is selective enough or by means of a full table scan if it is not highly selective.
It will then use the index on ``departments`` to access its data by ROWID, thereby joining it to the data from the leading row source.
When the filter on ``last_name`` is not as selective, especially when the cardinality of the ``departments`` table is lower than the cardinality of the ``employees`` table *after* the filter has been applied, the roles of driving and probe row sources are reversed.
 
If no such indexes exist at all, then a hash join seems logical.
Whether the ``departments`` or ``employees`` table is used to generate an in-memory hash cluster depends on what table Oracle believes will be best based on the cardinality estimates available.
We basically have the same logic as before: for highly selective filters, Oracle will use the ``employees`` table as the driving row source, otherwise it will pick (on) the ``departments`` table to take the lead.
 
Queries 2 and 3 yield different result sets, so it's more or less comparing apples and oranges.
Nevertheless, with an appropriate, selective index on ``last_name`` Oracle will probably settle for nested loops for Query 2 (i.e. the one with the ``WHERE`` clause), and a hash join for Query 3 (i.e. the one with the ``ON`` clause).
If the index on ``last_name`` is not selective at all and its clustering factor is closer to the number of rows than the number of blocks, then Query 2 may also be executed with a hash join, as we have discussed earlier.
Should the SQL engine decide on nested loops for Query 3, it is to be expected that the ``departments`` table be promoted to the position of driving row source because Oracle can use the single-column join condition on ``last_name`` as an access predicate.
 
Please note that a sort-merge join is possible in all instances.
The sort-merge join is rarely Oracle's first choice when faced with equality join conditions, especially when the tables involved are not sorted to start with.
 
So, what you should take away from this section is that even though the ``WHERE`` clause is technically a *post*-join filter, it can be and often is used by Oracle when it fetches the data of the leading row source, analogously to single-column predicates specified in the ``ON`` clause, thereby reducing the number of main iterations (i.e. over the driving row source) or the number of index lookups (in the probe row source) for nested loops, or the size of the in-memory hash cluster for a hash join.
For sort-merge joins these predicates can be used to minimize the size of the tables to be sorted, if one or both tables require reordering.
  
.. _lateral view: http://optimizermagic.blogspot.de/2007/12/outerjoins-in-oracle.html
.. _this thread: http://stackoverflow.com/a/7892125
.. _pipelining: http://sql-performance-explained.com
.. _left-deep join tree: http://www.oaktable.net/content/right-deep-left-deep-and-bushy-joins
.. _Expert Oracle SQL: http://www.apress.com/9781430259770
.. _Bushy joins: http://tonyhasler.wordpress.com/2008/12/27/bushy-joins
.. _full partition-wise join: http://docs.oracle.com/database/121/VLDBG/to_vldbg1348_d471.htm
.. _partial partition-wise join: http://docs.oracle.com/database/121/VLDBG/to_vldbg1349_d472.htm
.. _the number of partitions in relation to the DOP: http://blogs.oracle.com/datawarehousing/entry/partition_wise_joins
.. _generally recommended: https://docs.oracle.com/database/121/VLDBG/to_vldbg1353_d476.htm
