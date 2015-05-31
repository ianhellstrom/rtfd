.. _sql-hints-types-misc:

Miscellaneous Hints
-------------------
This category contains both documented and undocumented hints.
The ones we describe below are by no means meant to be an exhaustive list.
We have grouped them by topic for your convenience.

.. _sql-hints-dpins:

Direct-Path Inserts
^^^^^^^^^^^^^^^^^^^
A direct-path insert is an ``INSERT`` that stores data from the high-water mark (HWM) onward irrespective of the space available in the blocks below the HWM.
The advantage of a direct-path insert is that Oracle does not have to check whether any blocks below the HWM are available.
If a table is set to ``NOLOGGING``, then Oracle `minimizes redo generation`_, which means that a direct-path insert is generally faster than a regular insert.

For tables that data is never deleted from, this is fine, as there probably is no space below the HWM anyway.
When a table does have ample space below the HWM because of occasional ``DELETE`` statements, which do not cause the HWM to drop, the table may take up (and waste) `a lot of space`_, even if it contains very little data, as the HWM is gradually moved up with each direct-path insert and never dropped.
This in turn may significantly (negatively) affect the performance of queries against that table.
With ``TRUNC`` the HWM is always dropped to the lowest level possible, which is best in conjunction with direct-path inserts.

Since a direct-path insert is basically the same as appending data, the hint is named accordingly: ``APPEND``.
This hint is used for ``INSERT INTO ... SELECT`` statements, whereas the ``APPEND_VALUES`` hint is for ``INSERT INTO ... VALUES`` statements.
``NOAPPEND`` -- without an underscore! -- makes sure that the data is not inserted by means of a direct-path insert.
These hints do not affect anything other than ``INSERT`` statements.
How space is managed during a direct-path insert is described in detail on `the official optimizer blog`_.

Important to know is that during direct-path inserts certain constraints are disabled.
Only ``NOT NULL`` and ``UNIQUE`` (hence also ``PRIMARY KEY``) constraints remain *enabled*.
Rows that violate ``UNIQUE`` constraints are, however, `still loaded`_, which is different from the normal behaviour, where such rows are rejected. 

Not all tables can use a direct-path insert though.
In particular, clustered tables, tables with :term:`VPD` policies, tables with ``BFILE`` columns.
Similarly, direct-path insert is not possible on a *single partition* of a table if it has global indexes defined on it, referential (i.e. foreign-key) and/or check constraints, or *enabled* triggers defined.
Furthermore, no segments of a table can have open transactions.

What about partial deletions that cannot be simply ``TRUNC``'d?
The best solution is to partition the table and `drop entire partitions`_.
Beware that ``TRUNC`` is a DDL statement, which means that it comes with an implicit ``COMMIT`` in contrast to ``DELETE``, which is a DML statement and requires an explicit ``COMMIT`` (or ``ROLLBACK``).

Caching vs Materialization
^^^^^^^^^^^^^^^^^^^^^^^^^^
When Oracle performs a full table scan, it can place the blocks retrieved in the buffer cache, so that other SQL statements can benefit from the cached data.
This can be accomplished with the ``CACHE( tab_name_or_alias )`` hint, which typically has to be supplemented with the ``FULL( tab_name_or_alias )`` hint to ensure a full table scan is used.
Because this only works for full table scans and the buffer cache is limited in size, this is often best for small lookup tables.
The data is placed at the *most* recently used end of the `least recently used`_ (LRU) list in the buffer cache, which is Oracle's way of saying that the blocks line up for a LIFO queue.
``NOCACHE`` -- again, no underscore -- puts the blocks retrieved at the *least* recently used end of the LRU, which is the default and in most cases means that the data is discarded from the cache almost immediately.

Results of a query or query fragment, including those that are not obtained by means of a full table scan, can be cached with the ``RESULT_CACHE`` hint.
The hint can be placed at the top level, in a factored subquery, or an inline view.
Subsequent executions of the same statement can be satisfied with data from the cache, which means that Oracle can save on a few round trips to the database.
Cached results are automatically invalidated when a database object upon which the result depends is modified.

It is also possible to use system or session settings and/or table annotations to enable the :ref:`result cache <plsql-cache-alt-rc>`.
Typically the initialization parameter ``RESULT_CACHE_MODE`` is set to ``MANUAL``, as ``FORCE`` causes all statements' results to be cached, which is a bad idea when set at the system level.
The ``RESULT_CACHE`` attribute of tables is set to either the ``FORCE`` or ``DEFAULT`` mode.
``DEFAULT`` requires the ``RESULT_CACHE`` hint in all queries where the results should be cached, and because it is the default often requires no further action.
In case a table is set to ``FORCE`` mode, the ``NO_RESULT_CACHE`` hint can be used to override this behaviour for individual queries.
Table annotations apply to entire queries that reference these tables, not just individual query blocks.

Read consistency requires that whenever a session transaction references tables or views in query, the results from this query are not cached.
Furthermore, any (user-defined) functions used in the query have to be :ref:`deterministic <plsql-cache-alt-det>`, and the query may not contain temporary tables, tables owned by ``SYS`` or ``SYSTEM``, the ``CURRVAL`` or ``NEXTVAL`` pseudocolumns, or instantaneous time functions such ``SYSDATE`` or ``SYS_TIMESTAMP``.

There is also an undocumented ``MATERIALIZE`` hint that causes `factored subqueries to be materialized`_, that is they are stored in `a global temporary table`_ that is created (and dropped) on the fly.
Whenever a factored subquery is accessed more than once in the same SQL statement, the factored subquery in question is automatically materialized.

You can use the ``INLINE`` hint on factored subqueries to prevent the optimizer from materializing them.
This inlining can be useful when the data of a factored subquery is accessed several times but based on disjoint predicates from the main query that combines these intermediate results with ``UNION ALL``.
When the factored subquery is materialized, which would be the default behaviour in this case, Oracle cannot push a common predicate into the view because the predicates are disjoint.
This means that the factored subquery is evaluated for all possible values, materialized, and only then filtered accordingly.
With the ``INLINE`` hint you can prevent the materialization, which in turn means that Oracle can eliminate partitions, if the underlying tables are partitioned appropriately, or access data through indexes, meaning that it does not have to compute the factored subquery for all values *before* it filters.

Manual Cardinality Estimates
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
As we have said before, the cardinality is simply the number of rows divided by the number of distinct values (:math:`\mathit{NDV}`); a rough estimate of the selectivity is :math:`1/\mathit{NDV}`.
The cardinality is in all but heavily hinted SQL statements one of the top measures that influences the cost and thus the execution plan to be taken.
Consequently, accurate statistics are essential.

The optimizer is exceptionally good at its job, especially if it has all the data it needs.
That is also exactly the point: Oracle runs into problems when it either has no information or the information is not representative.
A case where Oracle has no real information is when it joins data with the data from a (pipelined) table function.
Oracle guesses the cardinality of a (pipelined) table function `based on the block size`_, which is perfectly fine for simple queries.
It gets tricky when you join the table function to other database objects, as now the cardinality affects the execution plan.
By the way, in case you are not familiar with table functions, you have already seen one example: ``SELECT * FROM TABLE( DBMS_XPLAN.DISPLAY )``.

An undocumented yet often used hint to aid Oracle when statistics are unavailable or inaccurate is ``CARDINALITY( tab_name_or_alias  number_of_rows )``.
It instructs the optimizer to treat the integer ``number_of_rows`` as the cardinality estimate of the table (function) ``tab_name_or_alias`` without actually checking it.

Whether the ``CARDINALITY`` hint is safe or rather very dangerous depends on whether you subscribe to `Tom Kyte's`_ or `Tony Hasler's`_ (p. 479) views.
Changing the cardinality estimate is one of the surest ways to affect the execution plan, and, when used without caution and due diligence, can lead to sub-optimal or even horrible execution plans.

Another undocumented hint that serves a similar purpose is ``OPT_ESTIMATE( TABLE  tab_name_or_alias  SCALE_ROWS = scaling_factor )``.
You have to supply ``tab_name_or_alias`` and the ``scaling_factor``, which is a correction (or fudge) factor to scale the optimizer's estimates up or down.
The cardinality estimate used by the optimizer is thus the original estimate times the scaling factor.

There is also a variation on ``OPT_ESTIMATE`` that works exactly like ``CARDINALITY``: ``OPT_ESTIMATE( TABLE  tab_name_or_alias  ROWS = number_of_rows )``.
The main advantage of the ``OPT_ESTIMATE`` hint is its `versatility`_.
We can also use it to specify an estimate of the number of rows returned from a join: ``OPT_ESTIMATE( JOIN  (tab_name_or_alias, another_tab_name_or_alias)  ROWS = number_of_rows )``.

In addition, there is the ``DYNAMIC_SAMPLING( tab_name_or_alias  sampling_level )`` hint for (pipelined) table functions.
When you set ``sampling_level`` to 2 or higher for pipelined table functions, a `full sample`_ of the row source is *always* taken.

Alternatively, you can use the `extensible optimizer`_ or rely on cardinality feedback, which is also known as statistics feedback.
For cardinality feedback it is important to note that on 11gR2, the feedback was lost once the statement departed from the shared pool.
From 12c onwards, the cardinality feedback is still available in the ``SYSAUX`` tablespace.

Distributed Queries
^^^^^^^^^^^^^^^^^^^
Distributed queries access data from at least one remote data source.
To decrease overall I/O and thus improve the performance of the execution of a SQL statement, you want to minimize the amount of data to be moved around, especially across the database link.
With ``DRIVING_SITE( tab_name_or_alias )`` you tell the optimizer to use the database in which ``tab_name_or_alias`` resides as the location to do all operations in; all data that is required to execute the statement is moved to that database through database links emanating from the initiating (local) database to the remote data sources.
This hint may be required because the local database `may not have access to statistics on the remote site(s)`_.
Oracle only chooses a remote database without the ``DRIVING_SITE`` hint when *all* the row sources are at that site.

You typically use this hint to instruct Oracle to choose the database with the largest amount of data as the driving site.
What you have to be aware of are user-defined PL/SQL functions that are on a different site than the driving site, as they cause a sizeable performance hit because of data ping-pong.
Similarly, beware of sort operations on the final result set as they are taken care of by the local database.
`Ian Hellström`_ has described some of the issues with distributed queries in more detail.

Join Transformations
^^^^^^^^^^^^^^^^^^^^
Sometimes Oracle can eliminate a join when querying from views.
This can happen when you query a view that joins two or more tables but you only ask for data from one of the tables involved in the view's join.
Oracle can automatically do a `join elimination`_ in these cases but it is also able to do so when referential integrity (i.e. a foreign-key constraint) guarantees that it is OK to do so.

For instance, the child table is the one we're mainly interested in but we would also like to have data from the parent table that is linked to the child table's data by means of a foreign key.
Oracle now *knows* that it can simply obtain the information from the child table because referential integrity guarantees that any reference to the parent's column(s) can be replaced by a corresponding reference to the child's  column(s).
What often cause Oracle to miss referential integrity constraints and thus the join elimination are aggregate functions, as it may not be clear to the optimizer that each row in the child table has exactly one matching row in the parent table.
If that is the case, it may often help to rewrite the join such that Oracle can be sure that the integrity is preserved: a simple left-join of child table to its parent will do the trick.

When you have ensured referential integrity with a foreign-key constraint, a join elimination (default) can be forced with the ``ELIMINATE_JOIN( tab_name_or_alias )`` or disabled with ``NO_ELIMINATE_JOIN ( tab_name_or_alias )``.
The parameter ``tab_name_or_alias`` can be either a (parent) table or alias thereof, or a space-separated list of tables or aliases thereof.

There are also instances when an outer join can be transformed to an inner join without affecting the result set because of ``IS NOT NULL``-predicates on the columns in the independent (parent) table, which are referred to by the dependent (child) tables in foreign-key constraints.
Oracle does this automatically but it can be enabled (disabled) with ``OUTER_JOIN_TO_INNER( tab_name_or_alias )`` (``NO_OUTER_JOIN_TO_INNER( tab_name_or_alias )``).
Again, the parameter ``tab_name_or_alias`` can be either a (parent) table or alias thereof, or a space-separated list of tables or aliases thereof.

There is an analogous hint for a conversion from a full outer join to an outer join: ``FULL_OUTER_JOIN_TO_OUTER( tab_name_or_alias )``, where ``tab_name_or_alias`` is the (parent) table with a ``IS NOT NULL``-predicate (or similar).

The last transformation that we wish to discuss in this group is the semi-to-inner join transformation with its hints ``SEMI_TO_INNER( tab_name_or_alias )`` and ``NO_SEMI_TO_INNER( tab_name_or_alias )``.
It applies to subqueries in ``EXISTS``-predicates and it causes the nested (correlated) subqueries to be joined to the main query as separate inline views.

How is this different from subquery unnesting?
Good question!
After a subquery has been unnested, the previously nested subquery always becomes the probe row source.
With a semi-to-inner join transformation this subquery can also be used as the driving row source.

Predicate and Subquery Push-Downs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The ``PUSH_PRED( tab_name_or_alias )`` hint can be used to push a join predicate into an inline view, thus making the inline view a correlated subquery.
As a consequence, the `subquery must be evaluated for each row of the main query`_, which may not sound like a good idea until you realize that it enables the optimizer to access the base tables and views in the inline view through indexes in nested loops.

When the main query returns many rows this transformation rarely leads to an optimal plan.
The optimizer typically considers this transformation based on cost but if you believe the optimizer to be mistaken in its decision to discard this transformation, you can provide the hint.

A join predicate push-down (JPPD) transformation can be applied when the inline view is one of the following:

* A ``UNION [ ALL ]`` view.
* An outer-joined, anti-joined, or semi-joined view.
* A ``DISTINCT`` view.
* A ``GROUP BY`` view.

Compare this to the list of when `view merging is not possible`_:

* When the view contains any of the following constructs:

  * an outer join;
  * set operators (e.g. ``UNION ALL``);
  * ``CONNECT BY``;
  * ``DISTINCT``;
  * ``GROUP BY``.

* When the view appears on the right-hand side of an anti- or semi-join.
* When the view contains scalar subqueries in the ``SELECT``-list.
* When the outer query block contains PL/SQL functions.

When using the ``PUSH_PRED`` hint you also have to supply ``NO_MERGE`` to prevent the inline view from being merged into the main query, although -- as you can see from the aforementioned criteria -- view merging and JPPD are generally mutually exclusive.
Notably absent from the list of inline views that allow a JPPD is the inner join, which means that *if* you believe a JPPD to be favourable *and* the optimizer does not already consider it to yield the optimal execution plan, you may have to convert an inner to an outer join, just to allow the JPPD transformation.

The execution plan contains an operation ``PUSHED PREDICATE`` when the JPPD is successfully applied.
``NO_PUSH_PRED`` does exactly the opposite of ``PUSH_PRED``.

The optimizer can also evaluate non-merged or non-unnested (i.e. nested) subqueries as soon as possible.
Usually such subqueries are evaluated as the last step, but it may be useful to `favour the subquery earlier in the process`_, for instance because its evaluation is relatively inexpensive and reduces the overall number of rows considerably.
The ``PUSH_SUBQ`` hint can be used to that end.
It is recommended that you specify the `query block as a parameter`_, because as of 10g this hint can be applied to `individual subqueries rather than all subqueries`_.
When you apply the hint to a remote table or a table that is joined with a sort-merge join, it has no effect.
There is of course also a ``NO_PUSH_SUBQ`` to disable subquery push-downs.

The ``PRECOMPUTE_SUBQUERY`` hint is related but not the same; it applies to ``IN``-list subqueries.
In fact, it instructs the optimizer to isolate the subquery specified with a `global temporary table`_.
The results from this separate execution are then `hard-coded into the main query`_ as filter values.

Set-Operation Transformations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Set transformation hints that have been `deprecated`_, such as ``HASH_XJ``, ``MERGE_XJ``, and ``NL_XJ``, where ``X`` can be either ``S`` for semi-joins or ``A`` for anti-joins, are not listed here.
One set-operation transformation that appears to have slipped through the cracks of deprecation is the ``SET_TO_JOIN( @SET$N )`` with ``N`` the identifier of the set.
It can be used to transform queries with ``MINUS`` and ``INTERSECT`` to their equivalents with joins.
Without the hint the optimizer *never* considers the set-to-join transformation.

In case the initialization parameter ``_CONVERT_SET_TO_JOIN`` has been set, you can use ``NO_SET_TO_JOIN( @SET$N )`` to disable the transformation.

.. _`minimizes redo generation`: http://oracle-base.com/articles/misc/append-hint.php#how-the-append-affects-the-table-size-high-water-mark
.. _`a lot of space`: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::p11_question_id:1951476814728
.. _`drop entire partitions`: http://ora600tom.wordpress.com/2012/05/30/append-hint-and-table-space-management
.. _`least recently used`: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:7828371300346568672
.. _`factored subqueries to be materialized`: http://www.dba-oracle.com/t_materialize_sql_hint.htm
.. _`a global temporary table`: http://oracle-base.com/articles/misc/with-clause.php#materialize-hint
.. _`based on the block size`: http://oracle-base.com/articles/misc/pipelined-table-functions.php#cardinality
.. _`extensible optimizer`: http://www.oracle-developer.net/display.php?id=427
.. _`Tom Kyte's`: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:2233040800346569775
.. _`Tony Hasler's`: http://www.apress.com/9781430259770
.. _`versatility`: http://www.pythian.com/blog/oracles-opt_estimate-hint-usage-guide
.. _`full sample`: http://www.oracle-developer.net/display.php?id=427
.. _`subquery must be evaluated for each row of the main query`: http://blogs.oracle.com/optimizer/entry/basics_of_join_predicate_pushdown_in_oracle
.. _`view merging is not possible`: http://blogs.oracle.com/optimizer/entry/optimizer_transformations_view_merging_part_1
.. _`favour the subquery earlier in the process`: http://www.morganslibrary.org/reference/hints.html
.. _`individual subqueries rather than all subqueries`: http://jonathanlewis.wordpress.com/2007/03/09/push_subq
.. _`query block as a parameter`: http://www.dba-oracle.com/t_push_subq_hint.htm
.. _`global temporary table`: http://www.dba-oracle.com/t_precompute_subquery_hint.htm
.. _`hard-coded into the main query`: http://blog.tanelpoder.com/2009/01/23/multipart-cursor-subexecution-and-precompute_subquery-hint
.. _`may not have access to statistics on the remote site(s)`: http://jonathanlewis.wordpress.com/2013/08/19/distributed-queries-3
.. _`Ian Hellström`: http://wp.me/p4zRKC-3b
.. _`deprecated`: http://docs.oracle.com/cd/B12037_01/server.101/b10752/whatsnew.htm
.. _`join elimination`: http://oracle-base.com/articles/misc/join-elimination.php
.. _`still loaded`: http://docs.oracle.com/database/121/SUTIL/ldr_modes.htm#SUTIL1331
.. _`the official optimizer blog`: http://blogs.oracle.com/optimizer/entry/space_management_and_oracle_direct
