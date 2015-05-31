.. _sql-hints-types-transformation:

Query Transformation Hints
--------------------------
Again, all hints in this category, save for the generic ``NO_QUERY_TRANSFORMATION`` hint, come in couples:

* ``FACT`` / ``NO_FACT``
* ``MERGE`` / ``NO_MERGE``
* ``NO_EXPAND`` / ``USE_CONCAT``
* ``REWRITE`` / ``NO_REWRITE``
* ``UNNEST`` / ``NO_UNNEST``
* ``STAR_TRANSFORMATION`` / ``NO_STAR_TRANSFORMATION``

With ``NO_QUERY_TRANSFORMATION`` you disable all query transformation that the optimizer can perform.
What the hint does *not* disable, though, are transformations that the optimizer *always* applies, such as the count transformation, predicate move-around, filter push-down, distinct elimination, and select-list pruning.
This is of course no hint for a production environment, and it should only be used for testing purposes.

Generic Transformations
^^^^^^^^^^^^^^^^^^^^^^^
We have already briefly seen the ``NO_MERGE( view_name_or_alias )`` hint.
It prohibits the optimizer from merging views in a query.
Similarly, you can force Oracle to merge (inline) views with ``MERGE( view_name )``.

.. note::
   ``MERGE`` and ``NO_MERGE`` have nothing to do with the sort-merge join!

When the view contains a ``GROUP BY`` clause or ``DISTINCT`` operator (or ``UNIQUE``) operator, the ``MERGE`` hint only works if `complex view merging`_ is enabled.
The delayed evaluation of these operations can either improve or worsen performance, so use these hints wisely and sparingly.
For instance, join conditions may reduce the data volume to be grouped or sorted, which may be beneficial to performance.
Likewise, it can be advantageous to aggregate data as early as possible to deal with less data in subsequent operations.
The optimizer uses the cost to determine whether it is better to merge views or not.
Complex view merging also allows uncorrelated ``IN``-subqueries to be merged into the main SQL statement.

``USE_CONCAT`` always enables the ``OR``-expansion, which transforms combined ``OR``-conditions or ``IN``-lists in the ``WHERE`` clause into a compound query with the ``UNION ALL`` operator.
Whether the cost with such an ``OR``-expansion is truly lower than without it is irrelevant: when specified Oracle does as instructed.
``NO_EXPAND`` makes the optimizer discard the ``OR``-expansion as a possible query transformation.

Subquery unnesting can be forced without regard for the cost with the ``UNNEST`` hint.
It combines subqueries in the ``WHERE``, such as in ``IN``-lists, into the ``FROM`` clause, which opens the door to more access paths for the optimizer to tinker with.
Without subquery unnesting, Oracle treats the main query and its subqueries as separate statements: the subqueries are executed, and their results are used to run the main query.
Subquery unnesting is possible if and only if the resulting join statement is guaranteed to returns the same rows as the original statement, for instance thanks to a primary key, and the subquery does not contain any aggregate functions.
``NO_UNNEST`` is, as you may have guessed, used to disable subquery unnesting.
Oracle unnests subqueries automatically unless hinted, regardless of cost expected.

Materialized views that have been created with the ``ENABLE QUERY REWRITE`` clause can be used to provide data to queries that do not explicitly call these materialized view in their ``FROM`` clauses.
Contrary to regular views, which are nothing but stored queries, materialized views store the result sets of the queries that define them and regularly refresh the data.
Materialized views are particularly useful for queries that are run often, as a snapshot of the data is taken and stored, so the data does not have to be calculated from scratch every time a user asks for it.
However, some users may not be aware of these materialized views, which is why they are executing their own queries that ask for the same data as contained in the materialized views.
With ``REWRITE`` you allow people to benefit from the data of these materialized views; the hint has an optional parameter, which is the name of the materialized view.
Typically, Oracle does this automatically when it determines that such a `rewrite`_ is beneficial.
If successful, it shows up in the execution plan as ``MAT_VIEW REWRITE``.

``NO_REWRITE`` overrides the ``ENABLE QUERY REWRITE`` clause, if present.
This can be helpful if you know that the data in the materialized view is stale compared to the source tables, and your query needs the current state of affairs.

The Star Transformation
^^^^^^^^^^^^^^^^^^^^^^^
In many data warehouses and OLAP databases that power business intelligence solutions, the dimensional rather than the entity-relationship data model is the gold standard. [#datamodels]_
Fact tables contain all information pertinent to a user's queries, and they can easily be joined to so-called dimension tables with more details on the dimensions listed in the fact table.
The schema for such databases resembles what we refer to as a snowflake schema.
In such instances, a star transformation can be useful, to which end Oracle has introduced the ``STAR_TRANSFORMATION`` hint.
When specified, Oracle does not guarantee that it will be used.

A requirement for the star transformation is that there be a `single-column bitmap index on all foreign-key columns of the fact table`_ that participate in the join.
The star transformation progresses in two steps:

#. Transform the original query with the join into a query with the fact table in the ``FROM`` clause and the dimension tables as subqueries in the ``WHERE`` clause to filter rows from the fact table based on the dimensions' values or ranges.
   The bitmap indexes are then combined with the bitmap ``AND``-operation to select only the rows that satisfy all dimensional constraints.
   The advantage is that all the dimension tables are logically joined with the fact table only once rather than once for each dimension table.
   Which join method is used depends on what the optimizer decides is best, although typically for large data volumes a hash join is chosen.
#. Adjoin the rows from the dimension tables to the fact table using the best access method available to the optimizer, which is typically a full table scan because dimension tables are often relatively small.

We have said that Oracle does not always perform a star transformation, even though the ``STAR_TRANSFORMATION`` hint is specified.
This is even true when all prerequisites, such as said bitmap indexes on the fact table, are met.
In fact, the optimizer calculates the best plan without the transformation and only then compares it to the best plan with the transformation.
Based on the costs of both plans, it picks one, which may not always be the one with the transformation enabled.
One such case is when a large fraction of the rows in the fact table need to be fetched, for instance because the constraints on the dimension tables are not selective enough.
It is then often advantageous to do a full table scan with multi-block reads.

Most of the time, database developers are told that bind variables are the key to great performance.
When your query has bind variables, the star transformation will never be used though.

Another instance when star transformations are never applied is when *fact* tables are accessed remotely, that is through a database link.
Dimension tables may, however, be on different Oracle database instances.

Anti-joins, fact tables that are unmerged or partitioned views, and dimension tables that appear both in the ``FROM`` clause and as subqueries in the ``WHERE`` clause are a few other party poopers for the star transformation.

The ``FACT( tab_name_or_alias )`` hint can be used to inform the optimizer which table should be considered the fact table.
``NO_FACT`` is exactly the opposite.

.. _`complex view merging`: http://blogs.oracle.com/optimizer/entry/optimizer_transformations_view_merging_part_2
.. _`single-column bitmap index on all foreign-key columns of the fact table`: http://docs.oracle.com/cd/B19306_01/server.102/b14223/schemas.htm#CIHGHEFB
.. _`rewrite`: http://docs.oracle.com/database/121/DWHSG/qrbasic.htm#DWHSG018

.. rubric:: Notes

.. [#datamodels] We have no intention of starting a debate on the data model paradigms of Kimball and Inmon. The interested reader will find plenty of insightful articles `on the internet <http://searchbusinessintelligence.techtarget.in/tip/Inmon-vs-Kimball-Which-approach-is-suitable-for-your-data-warehouse>`_.
