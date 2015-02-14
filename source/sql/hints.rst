.. _sql-hints:

*****
Hints
*****
According to the `Oxford Dictionary`_, a hint is "a slight or indirect indication or suggestion".
Oracle optimizer hints can be broadly categorized in two classes: 1) real hints, and 2) instructions.
The former, real hints, match the dictionary definition: they provide Oracle with pertinent information that it may not have when it executes a particular statement.
The information provided manually, for instance a cardinality estimate of an object, may aid in the search for an optimal execution plan.
The latter are really best called `instructions`_ or directives as they tell the optimizer what (not) to do.
 
Hints are actually comments in SQL statements that are read by the optimizer.
They are indicated by the plus sign in ``/*+ SOME_HINT */``.
Without the plus sign the comment would be just that: a comment.
And the optimizer does not care about the comments you add to increase the quality or legibility of your code.
With the plus sign, the optimizer uses it do determine the execution plan for the statement.
 
As always, the hint may be written in any case (UPPER, lower, or miXEd) but it must be valid for the optimizer to be able to do anything with it.
Whether you add a space after the '``+``' and/or before the closing '``*/``' is up to you; Oracle does not care either way.
 
It is customary to place the hint directly after the SQL verb (e.g. ``SELECT``) but it is not necessary to do so.
However, the hint must follow the plus sign for the optimizer to pick it up, so do not place any comments in front of the hint.
In fact, we recommend that you do not mix comments and hints anyway.
Several hints may be placed in the same 'comment'; they have to be separated by (at least) one space, and they may be on different lines.
 
Mind you, the syntax checker does *not* tell you whether a hint is written correctly and thus available to the optimizer!
You're on your own there.
If it's not valid, it's simply ignored.
 
We shall henceforth use upper-case hints, lower-case parameters that have to be provided by the developer, and separate all parameters by two spaces to make it easier for you to read the hints.
Some people prefer to use commas between parameter values but that is not strictly necessary, so we shall leave out the commas.
 
When To Use Hints
=================
Oracle recommends that "hints [...] be used sparingly, and only *after you have collected statistics on the relevant tables and evaluated the optimizer plan without hints* using the ``EXPLAIN PLAN`` statement."
We have added the emphasis because that specific phrase is critical: don't add hints because you *think* you know better than the optimizer.
Unless you are an Oracle virtuoso you probably do not understand the fine print of each hint and its impact on the performance of your statements -- we certainly don't.
 
Still, you may have good reasons to add hints to your statements.
We have listed some of these reasons below.
 
* For demonstrative purposes.
* To try out different execution plans in the development phase.
* To provide Oracle with pertinent information when no such information is available in statistics.
* To fix the execution plan when you are absolutely sure that the hints lead to a significant performance boost.
 
There may be more, but these four sum up the whys and wherefores pretty well.
 
When Not To Use Hints
=====================
Even though Oracle -- ahem -- hints us to use hints as a last resort, Oracle whizz `Jonathan Lewis goes even further`_ and pretty much has one simple rule for hints: don't.
 
We do not take such a grim view of the world of hints but do wish to point out that there are actually very good reasons not to use hints at all.
 
A common use case for hints is when statistics are out of date.
In such cases hints can indeed be useful, but an approach where representative statistics are locked, such as the TSTATS approach, may be warranted and in fact more predictable.
Hints are basically static comments that fiddle around with the internals of the optimizer.
Dynamically generated hints (i.e. hints generated on-the-fly in dynamic SQL) are *extremely* rare, and come to think of it, we have never seen, heard of, or read about them anywhere.
Full stop. [#dynhints]_
 
Because hints are static they are in some ways the same as locked statistics.
The only difference is that once you have seen (in development) that different statistics are more appropriate and you need to do a refresh, you can release the statistics into production by copying the statistics from development to production.
Oracle will take care of the rest, and the optimizer can figure out the new best execution plan on its own, which will be the same as in development because the statistics are identical.
You really don't want to run into performance surprises in production, especially if they suck the life out of your statements.
 
With hints, you have to check the performance against both the development and the production instance(s), as the statistics may very well be different.
What you lose with hints is the predictability of locked statistics.
You also need to regularly verify that the hints still perform as initially intended.
And typically that is not something you want to do in a live system.
 
Moreover, when Oracle releases a new version of its flagship database, it usually comes packed with lots of improvements to the optimizer too.
If you have placed hints inside your code, the optimizer does what you tell it to do, and you are unlikely to benefit from any new features.
Some of these features may not always be helpful but in many cases they really are.
 
This brings us to another important point.
Hints should be revisited regularly and documented properly.
Without documentation no one except you during a period of a few hours or perhaps days, depending on nimbleness of your short-term memory and the amount of beer and pizza consumed in the time afterwards, will know why the hints were required at all, and why that particular combination of hints was chosen.
 
In summary, don't use hints when
 
* what the hint does is poorly understood, which is of course not limited to the (ab)use of hints;
* you have not looked at the root cause of bad SQL code and thus not yet tapped into the vast expertise and experience of your DBA in tuning the database;
* your statistics are out of date, and you can refresh the statistics more frequently or even fix the statistics to a representative state;
* you do not intend to check the correctness of the hints in your statements on a regular basis, which means that, when statistics change, the hint may be woefully inadequate;
* you have no intention of documenting the use of hints anyway.
 
Named Query Blocks
==================
You may have already seen object aliases in the execution plan.
An object alias is a concatenation of the name of an object or its alias and the name of the query block it appears in.
A `query block`_ is any inline view or subquery of a SQL statement.
 
Object aliases typically look something like ``tab_name@SEL$1``, ``tab_nameINS$2``, ``tab_nameUPD$3``, ``tab_name@DEL$4``, or ``tab_name@MISC$5`` .
These automatically generated names are hardly insightful, which is why you are allowed to name query blocks yourself.
 
You name query blocks with ``/*+ QB_NAME( your_custom_qb_name ) */``.
Afterwards you can reference objects from that named query block using ``@your_custom_qb_name  tab_name_or_alias``.
The optimizer will use the custom name instead of ``SEL$1`` or whatever is applicable, so you can more easily understand the execution plan's details.
Note that the optimizer ignores any hints that reference different query blocks with the same name.
 
Should you name all query blocks?
 
Hell no!
Only use the query block name hint when your statements are complex and you need to reference objects from various query blocks in your hints.
When would you want to do that?
When you use global hints.
 
Global Hints
============
Hints are commonly embedded in the statement that references the objects listed in the hints.
For hints on tables that appear inside views Oracle recommends using `global hints`_.
These hints are `not embedded in the view itself`_ but rather in the queries that run off the view, which means that the view is free of any hints that pertain to retrieving data from the view itself.
 
We shall presume that we have created a view called ``view_name``.
The view does a lot of interesting things but what we need for a global hint in our query that selects data from our view is a table ``tab_name`` inside a subquery (e.g. inline view or factored subquery) with the alias ``subquery_alias``.
We would then write ``SELECT /*+ SOME_HINT( view_name.subquery_alias.tab_name ) */ * FROM view_name``, where ``SOME_HINT`` is supposed to be any valid optimizer hint.
 
Similarly we could use a named query block to do the same: ``/*+ SOME_HINT( @my_qb_name  tab_name )``, where ``my_qb_name`` is the name we have given to the query block in which ``tab_name`` appears.
You can also use the automatically generated query block names but that is begging for trouble.
Named query blocks are really useful in conjunction with global hints.
 
Types of Hints
==============
Oracle has kindly provided `an alphabetical list`_ of all *documented* hints.
There are also a bunch of undocumented ones, and examples of their use can be found scattered all over the internet and in the multitude of books on Oracle performance tweaking.
Undocumented hints are not more dangerous than their documented equivalents; Oracle simply has not gotten round to documenting them yet.
 
Oracle classifies hints based on their function:
 
* Optimization goals and approaches;
* Access path hints;
* In-memory column store hints;
* Join order hints;
* Join operation hints;
* Parallel execution hints;
* Online application upgrade hints;
* Query tranformation hints;
* XML hints;
* Other hints.
 
In `Oracle Database 12c Performance Tuning Recipes`_, the authors provide two additional types of hints:
 
* Data warehousing hints;
* Optimizer hints.
 
The data warehousing hints are actually included in Oracle's query transformation hints.
 
Access path and query transformation hints are by far the largest two categories, save for the miscellaneous group.
 
Although interesting in their own way we shall not discuss in-memory column store hints, online application upgrade hints, and XML hints.
We shall now go through the remaining categories and discuss the most important hints for each category, so you too can supercharge your SQL statements.
There are many more hints than we describe here, and you are invited to check the official documentation for more hints and details.
 
Optimization Goals and Approaches
---------------------------------
Oracle only lists two hints in this category: ``ALL_ROWS`` and ``FIRST_ROWS( number_of_rows )``.
These are mutually exclusive.
If you happen to be drunk while programming and inadvertently write both hints in the same statement, Oracle will go with ``ALL_ROWS``.
 
In mathematical optimization nomenclature, these two hints affect the objective function.
``ALL_ROWS`` causes Oracle to optimize a statement for throughput, which is the minimum *total* resource consumption.
The ``FIRST_ROWS`` hint does not care about the throughput and instead chooses the execution plan that yields the first ``number_of_rows`` specified as quickly as possible.
 
Note that Oracle ignores ``FIRST_ROWS`` in all ``DELETE`` and ``UPDATE`` statements and in ``SELECT`` statement blocks that include sorts and/or groupings, as it needs to fetch all relevant data anyway.
 
Optimizer Hints
---------------
We have already mentioned the ``GATHER_PLAN_STATISTICS`` hint, which can be used to obtain statistics about the execution plan during the execution of a statement.
It is especially helpful when you intend to `diagnose performance issues`_ with a particular statement.
It is definitely not meant to be used in production instances!
 
There is also a ``GATHER_OPTIMIZER_STATISTICS``, which Oracle lists under 'Other hints'.
It can be used to collect bulk-load statistics for CTAS statements and ``INSERT INTO ... SELECT`` statements that use a direct-path insert, which is accomplished with the ``APPEND`` hint, but more on that later.
The opposite, ``NO_GATHER_OPTIMIZER_STATISTICS`` is also provided.
 
The ``OPTIMIZER_FEATURES_ENABLE`` hint can be used to *temporarily* disable certain (newer) optimizer feature after database upgrades.
This hint is typically employed as a short-term solution when a small subset of queries performs badly.
Valid parameter values are `listed in the official documentation`_.
 
Access Path Hints
-----------------
Access path hints determine how Oracle accesses the data you require.
They can be divided into two groups: access path hints for tables and access path hints for indexes.
 
Tables
^^^^^^
The most prominent hint in this group is the ``FULL( tab_name )`` hint.
It instructs the optimizer to access a table by means of a full table scan.
If the table you want Oracle to access with a full table scan has an alias in the SQL statement, you have to use the alias rather than the table name (without the schema name) as the parameter to ``FULL``.
For named query blocks you have to provide the query block's name as discussed previously.
 
In this group are also the ``CLUSTER`` and ``HASH`` hints, but they apply only to tables in an indexed cluster and hash clusters respectively.
 
Indexes
^^^^^^^
The hints in this group all come in pairs:
 
* ``INDEX`` / ``NO_INDEX``
* ``INDEX_ASC`` / ``INDEX_DESC``
* ``INDEX_FFS`` / ``NO_INDEX_FFS``
* ``INDEX_SS`` / ``NO_INDEX_SS``
* ``INDEX_SS_ASC`` / ``INDEX_SS_DESC``
* ``INDEX_COMBINE`` / ``INDEX_JOIN``
 
All these hints take at least one parameter: the table name or alias in the SQL statement.
A second parameter, the index name(s), is optional but often provided.
If more than one index is provided, the indexes are separated by at least one space; the ``INDEX_COMBINE`` hint is recommended for this use case though.
 
Let's get cracking.
The first pair instructs the optimizer to either use (or not use) an index scan on a particular table.
If a particular index is specified, then Oracle uses that index to scan the table.
If no index is specified and the table has more than one index, the optimizer picks the index that leads to the lowest cost when scanning the data.
These hints are valid for any function-based, domain, B-tree, bitmap, and bitmap join index.
 
Similarly, you can tell the optimizer that it needs to scan the specified index in ascending order with ``INDEX_ASC`` or descending order with ``INDEX_DESC`` for statements that use an index range scan.
Note that if your index is already in descending order, Oracle ignores the ``INDEX_DESC`` hint.
 
No, ``FFS`` does not stand for "for f*ck's sake".
Instead it indicates that Oracle use a fast full index scan instead of a full table scan.
 
An index skip scan can be enabled (disabled) with ``INDEX_SS`` (``NO_INDEX_SS``).
For index range scans, Oracle scans index entries in ascending order if the index is in ascending order and in descending order if the index is in descending order.
You can override the default scan order with the ``INDEX_SS_ASC`` and ``INDEX_SS_DESC`` hints.
 
The pair ``INDEX_COMBINE`` and ``INDEX_JOIN`` is the odd one out, as they are not each other's opposites.
``INDEX_COMBINE`` causes the optimizer to use a bitmap access path for the table specified as its parameter.
If no indexes are provided, the optimizer chooses whatever combination of indexes has the lowest cost for the table.
When the ``WHERE`` clause of a query contains several predicates that are covered by different bitmap indexes, this hint may provide superior performance, as bitmap indexes can be combined very efficiently.
If the indexes are not already bitmap indexes, Oracle will perform a ``BITMAP CONVERSION`` operation.
As Jonathan Lewis puts it in the comments section of `this blog post`_: it's a damage-control access path.
You generally would not want to rely on bitmap conversions to combine indexes; it is often much better to improve upon the index structure itself.
 
The ``INDEX_JOIN`` instructs the optimizer to join indexes (with a hash join) to access the data in the table specified.
You can only benefit from this hint when there is a *sufficiently* small number of indexes that contains all columns required to resolve the query.
Here, 'sufficiently' is Oraclespeak for as few as possible.
This hint is worth considering when your table has `many indexed columns but only few of them are referenced`_ (p. 560) in your statement.
In the unfortunate event that Oracle decides to join indexes and you are certain that that is not the optimal access path, you cannot directly disable it.
Instead you can use the ``INDEX`` hint with only one index or the ``FULL`` hint to perform a full table scan.
 
Join Order Hints
----------------
The optimizer lists all join orders to choose the best one.
What it does not do is an exhaustive search.
 
In case you believe a different join order to be useful, you can use one of the join order hints: ``ORDERED`` or ``LEADING``.
The latter is more versatile and should thus be preferred.
 
``ORDERED`` takes no parameters and instructs the optimizer to join the tables in the order as they appear in the ``FROM`` clause.
Because the ``ORDERED`` hint is so basic and you do not want to move around tables in the ``FROM`` clause, Oracle has provided us with the ``LEADING`` hint.
It takes the table names or aliases (if specified) as parameters, separated by spaces.
 
In the optimizer's rock-paper-scissors game, ``ORDERED`` beats ``LEADING`` when both are specified for the same statement.
Moreover, if two or more conflicting ``LEADING`` hints are provided, Oracle ignores all of them.
Similarly, any ``LEADING`` hints are thrown into the bin when they are incompatible with dependencies in the join graph.
 
Join Operation Hints
--------------------
Join operation hints are also paired:
 
* ``USE_HASH`` / ``NO_USE_HASH``
* ``USE_MERGE`` / ``NO_USE_MERGE``
* ``USE_NL`` / ``NO_USE_NL``
 
These hints allow you to instruct the optimizer to use a hash join, a sort-merge join, or nested loops, respectively.
 
Hash joins support input swapping, which we have discussed when we talked about :ref:`left-deep and right-deep join trees <sql-join-trees>`.
This can be accomplished with ``SWAP_JOIN_INPUTS`` or prohibited with ``NO_SWAP_JOIN_INPUTS``.
 
The left-deep join tree can be enforced with the following hints:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 5-7
 
   /*+ LEADING( t1 t2 t3 t4 )
       USE_HASH( t2 )
       USE_HASH( t3 )
       USE_HASH( t4 )
       NO_SWAP_JOIN_INPUTS( t2 )
       NO_SWAP_JOIN_INPUTS( t3 )
       NO_SWAP_JOIN_INPUTS( t4 ) */
 
We could have also written ``USE_HASH( t4 t3 t2 )`` instead of three separate hints.
 
So, how do we go from a left-deep join ( ( T1 →  T2 ) → T3 ) → T4  to a right-deep join T4 → ( T3 → ( T2 → T1 ) )?
Remember the steps we had to perform, especially the swaps?
The process to go from the left-deep join tree to the right-deep join tree is to swap the order in the following sequence: T4, T3, and T2.
We can thus obtain the right-deep join tree by taking the left-deep join tree as a template and providing the necessary swaps:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 5-7
 
   /*+ LEADING( t1 t2 t3 t4 )
       USE_HASH( t2 )
       USE_HASH( t3 )
       USE_HASH( t4 )
       SWAP_JOIN_INPUTS( t2 )
       SWAP_JOIN_INPUTS( t3 )
       SWAP_JOIN_INPUTS( t4 ) */
       
The ``LEADING`` hint refers to the situation *before* all the swaps.
Important to know is that the left-deep join tree is *always* the `starting point`_.
 
Oracle occasionally bumps into bushy trees when views cannot be merged.
Bushy trees can, however, be practical in what is sometimes referred to as a `snowstorm schema`_, but we shall not go into more details here.
In instances where a bushy join is known to be advantageous you may have to rewrite your query.
For example, you can force Oracle to perform the bushy join ( T1 → T2 ) → ( T3 → T4 ) by writing the query schematically as follows:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 6-14,16-24
 
   SELECT /* LEADING ( v12 v34 )
             USE_HASH( v34 )
             NO_SWAP_JOIN_INPUTS( v34 ) */
     *
   FROM
     (
       SELECT /*+ LEADING( t1 t2 )
                  NO_SWAP_JOIN_INPUTS( t2 )
                  USE_HASH( t2 )
                  NO_MERGE */
         *
       FROM 
         t1 NATURAL JOIN t2
      ) v12
   NATURAL JOIN
     (
       SELECT /*+ LEADING( t3 t4 )
                  NO_SWAP_JOIN_INPUTS( t4 )
                  USE_HASH( t4 )
                  NO_MERGE */
         *
       FROM
         t3 NATURAL JOIN t4
      ) v34
   ;
 
You may have noticed that we have sneaked in the ``NO_MERGE`` hint, which we shall describe in somewhat more detail below.
What is more, we have used a ``NATURAL JOIN`` to save space on the ``ON`` or ``USING`` clauses as they is immaterial to our discussion.
  
Can you force Oracle to do a bushy join without rewriting the query?
 
Unfortunately not.
The reason is that there is no combination of swaps to go from a left-deep join tree to any bushy join tree.
You can do it with a bunch of hints for a zigzag trees, because only some of the inputs are swapped, but bushy trees are a nut too tough to crack with hints alone.
 
When you use ``USE_MERGE`` or ``USE_NL`` it is best to provide the ``LEADING`` hint as well.
The table first listed in ``LEADING`` is generally the driving row source.
The (first) table specified in ``USE_NL`` is used as the probe row source or inner table.
The syntax is the same for the sort-merge join: whichever table is specified (first) is the inner table of the join.
For instance, the combination ``/*+ LEADING( t1 t2 t3 ) USE_NL( t2 t3 ) */`` causes the optimizer to take T1 as the driving row source and use nested loops to join T1 and T2.
Oracle then uses the result set of the join of T1 and T2 as the driving row source for the join with T3.
 
For nested loops there is also the alternative ``USE_NL_WITH_INDEX`` to instruct Oracle to use the specified table as the probe row source and use the specified index as the lookup.
The index key must be applicable to the join predicate.
 
Parallel Execution Hints
------------------------
Not all SQL statements can be run in parallel.
All DML statements, including subqueries, can be run in parallel, which means that multiple blocks can be selected, inserted, deleted, or updated simultaneously.
For parallelized DDL statements, multiple blocks are being created/altered and written in parallel.
The DDL statements that can be run in parallel are:
 
* ``CREATE INDEX``
* ``CREATE TABLE ... AS SELECT``
* ``ALTER INDEX ... REBUILD``
* ``ALTER INDEX ... [ REBUILD | SPLIT ] PARTITION``
* ``ALTER TABLE ... MOVE``
* ``ALTER TABLE ... [ MOVE | SPLIT | COALESCE ] PARTITION``
 
Note that for the CTAS statement it is possible to perform a parallel DML (i.e. ``SELECT``) operation but write the data to disk serially, which means that it is not a parallel DDL operation.
We do not intend to dwell on such technicalities though.
 
The parallel execution of DDL statements requires that it be enabled either at the level of the session or by specifying the appropriate ``PARALLEL`` clause for the statement.
When set manually for a table or index with the ``ALTER { TABLE | INDEX } obj_name PARALLEL dop`` , the degree of parallelism (DOP) used to be for both subsequent DDL *and* DML statements *prior to 12c*.
Beware of that trap!
Nevertheless, since this is a section on optimizer hints, we have no intention of delving into the specifics on non-hinted parallel execution.
 
As of 11gR2 Oracle has had the ``PARALLEL( dop )`` and ``NO_PARALLEL ( tab_name_or_alias )`` for individual statements rather than sessions or objects.
The degree of parallelism ``dop`` is optional, and if omitted Oracle computes it for you; the minimum degree of parallelism is 2.
The ``PARALLEL`` hint will cause all access paths than can use parallelism to use it; in essence, the hint authorizes the optimizer to use parallelism.
The hint can be supplied to the ``SELECT``, ``INSERT``, ``UPDATE``, ``DELETE``, or ``MERGE`` bits of a statement.
 
Instead of supplying ``dop``, you can also write a) ``DEFAULT``, which means that the DOP is equal to the number of CPUs available on all instances multiplied by the value of the ``PARALLEL_THREADS_PER_CPU`` initialization parameter, b) ``AUTO``, which causes the optimizer to decide on the degree of parallelism or whether to run the statement serially, or c) ``MANUAL``, for which the degree of parallelism is determined by the objects in the statement.
 
The ``PARALLEL`` hint can also be set for specific objects in a SQL statement as follows: ``PARALLEL( tab_name_or_alias  dop )``.
You may also provide ``DEFAULT`` as an alternative to ``dop``; its behaviour is identical to the statement-level's hint.
 
In `Expert Oracle SQL`_ (p.152) it is noted that when inserting data in parallel *before* committing causes subsequent selects to fail *until* the data is committed.
The reason is that a `direct path write`_ can sometimes be used by parallel DML statements, especially inserts.
The rows for a direct path write are not in the :term:`SGA` and must be read from disk.
However, before the data is committed there is no fresh data to read from disk!
 
The ``NO_PARALLEL`` hint overrides the ``PARALLEL`` parameter supplied at the creation or alteration of any table.
 
In a similar fashion you can instruct Oracle to scan index ranges in parallel with the ``PARALLEL_INDEX( tab_name_or_alias  index_name  dop )``.
With ``NO_PARALLEL_INDEX( tab_name_or_alias  index_name )`` you can disable parallel index range scans.
In both hints, ``index_name`` is optional.
 
With ``PQ_CONCURRENT_UNION`` you force the optimizer to process ``UNION [ ALL ]`` operations in parallel.
``NO_PQ_CONCURRENT_UNION`` disables concurrent processing of said set operations.
 
When the distribution of the values of the join keys for a parallel join is highly skewed because many rows have the same join key values, parallelizing a join can be troublesome as the load is not easily distributed among the query servers.
To that end Oracle introduced the ``PQ_SKEW( tab_name_or_alias )`` hint, which informs the optimizer of data skew in the join keys.
Oracle requires `a histogram on the join expression`_ as otherwise it will probe rows at random to discover the skew; it also seems that only single inner joins are supported.
Similarly, there is a ``NO_PQ_SKEW( tab_name_or_alias )`` to advise the optimizer that most rows do not share the same join keys.
In both hints, ``tab_name_or_alias`` is the hash join's probe row source.
 
CTAS and ``INSERT INTO ... SELECT`` statements' distribution of rows between producers and consumers can be controlled with the ``PQ_DISTRIBUTE( tab_name_or_alias  distribution )`` hint.
The value of ``distribution`` can be one of the following:
 
* ``NONE``: no distribution, which is ideal when there is no skew, so the overhead of distributing rows can be avoided. It is important to be aware that each query server munches between 512 KB and 1.5 MB (with compression) of :term:`PGA` memory.
* ``PARTITION``: rows are distributed from producers to consumers based on ``tab_name_or_alias``'s partition information, which is best used when producer and consumer operations cannot be combined, there are more partitions than query servers, and there is no skew across partitions.
* ``RANDOM``: rows are distributed from the consumers to the consumers in a round-robin fashion, which is applicable when the data is skewed.
* ``RANDOM_LOCAL``: rows are distributed from the consumers to the consumers on the same RAC node in a round-robin fashion, which eliminates inter-node communication.
 
For joins it is also possible to use the hint in a slightly modified form: ``PQ_DISTRIBUTE( tab_name_or_alias  outer_distribution  inner_distribution )``.
`All possible values`_ are summarized in the table below.
 
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
|``outer_distribution`` | ``inner_distribution`` | Explanation                                                           | Use Case                                                                  |
+=======================+========================+=======================================================================+===========================================================================+
| ``HASH``              | ``HASH``               | Rows of both tables are mapped with a hash function on the join keys. | Tables have comparable sizes and join uses hash-join or sort-merge join.  |
|                       |                        | Each query server performs the join between pair of resulting         |                                                                           |
|                       |                        | partitions.                                                           |                                                                           |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
| ``BROADCAST``         | ``NONE``               | Rows of *outer* table are broadcast to each query server; rows of     | *Outer* table is small compared to inner table: inner-table size          |
|                       |                        | inner table are partitioned randomly.                                 | multiplied by number of query servers must be greater than outer-table    |
|                       |                        |                                                                       | size.                                                                     |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
| ``NONE``              | ``BROADCAST``          | Rows of *inner* table are broadcast to each query server; rows of     | *Inner* table is small compared to outer table: inner-table size          |
|                       |                        | outer table are partitioned randomly.                                 | multiplied by number of query servers must be less than outer-table size. |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
| ``PARTITION``         | ``NONE``               | Rows of *outer* table are mapped using partitioning of inner table;   | Number of partitions of *outer* table is roughly equal to number of query |
|                       |                        | inner table must be partitioned on join keys.                         | servers.                                                                  |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
| ``NONE``              | ``PARTITION``          | Rows of *inner* table are mapped using partitioning of outer table;   | Number of partitions of *outer* table is roughly equal to number of query |
|                       |                        | outer table must be partitioned on join keys.                         | servers.                                                                  |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
| ``NONE``              | ``NONE``               | Each query server joins a pair of matching partitions.                | Tables are equipartitioned on join keys.                                  |
|                       |                        |                                                                       |                                                                           |
+-----------------------+------------------------+-----------------------------------------------------------------------+---------------------------------------------------------------------------+
 
Please note that the last entry corresponds to the full partition-wise join we talked about :ref:`earlier <sql-join-partitions>`.

Finally, we have ``PQ_FILTER``, which tells Oracle how to process rows for correlated subqueries.
The following table shows all four parameter values, how the rows on the left-hand side and right-hand side of the filter are processed, and when best to use a particular parameter.

+------------+-------------------------------+----------+-----------------------------------------------------------------------------------------------+
| Parameter  | LHS                           | RHS      | Use Case                                                                                      |
+============+===============================+==========+===============================================================================================+
| ``HASH``   | Parallel: hash distribution   | Serial   | No skew in LHS data distribution                                                              |
+------------+-------------------------------+----------+-----------------------------------------------------------------------------------------------+
| ``NONE``   | Parallel                      | Parallel | No skew in LHS data distribution *and* LHS distribution best avoided (e.g. many rows in LHS ) |
+------------+-------------------------------+----------+-----------------------------------------------------------------------------------------------+
| ``RANDOM`` | Parallel: random distribution | Serial   | Skew in LHS data distribution                                                                 |
+------------+-------------------------------+----------+-----------------------------------------------------------------------------------------------+
| ``SERIAL`` | Serial                        | Serial   | Overhead of parallelization too high (e.g. few rows in LHS)                                   |
+------------+-------------------------------+----------+-----------------------------------------------------------------------------------------------+

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
Please note that ``MERGE`` and ``NO_MERGE`` have nothing to do with the sort-merge join!

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

It is also possible to use system or session settings and/or table annotations to enable the result cache.
Typically the initialization parameter ``RESULT_CACHE_MODE`` is set to ``MANUAL``, as ``FORCE`` causes all statements' results to be cached, which is a bad idea when set at the system level.
The ``RESULT_CACHE`` attribute of tables is set to either the ``FORCE`` or ``DEFAULT`` mode.
``DEFAULT`` requires the ``RESULT_CACHE`` hint in all queries where the results should be cached, and because it is the default often requires no further action.
In case a table is set to ``FORCE`` mode, the ``NO_RESULT_CACHE`` hint can be used to override this behaviour for individual queries.
Table annotations apply to entire queries that reference these tables, not just individual query blocks.

Read consistency requires that whenever a session transaction references tables or views in query, the results from this query are not cached.
Furthermore, any (user-defined) functions used in the query have to be ``DETERMINISTIC``, and the query may not contain temporary tables, tables owned by ``SYS`` or ``SYSTEM``, the ``CURRVAL`` or ``NEXTVAL`` pseudocolumns, or instantaneous time functions such ``SYSDATE`` or ``SYS_TIMESTAMP``.

There is also an undocumented ``MATERIALIZE`` hint that causes `factored subqueries to be materialized`_, that is they are stored in `a global temporary table`_ that is created (and dropped) on the fly.
Whenever a factored subquery is accessed more than once in the same SQL statement, the factored subquery in question is automatically materialized.

You can use the ``INLINE`` hint on factored subqueries to prevent the optimizer from materializing them.
This inlining can be useful when the data of a factored subquery is accessed several times but based on disjoint predicates from the main query that combines these intermediate results with ``UNION ALL``.
When the factored subquery is materialized, which would be the default behaviour in this case, Oracle cannot push a common predicate into the view because the predicates are disjoint.
This means that the factored subquery is evaluated for all possible values, materialized, and only then filtered accordingly.
With the ``INLINE`` hint you can prevent the materialization, which in turn means that Oracle can eliminate partitions, if the underlying tables are partitioned appropriately, or access data through indexes, meaning that it does not have to compute the factored subquery for all values *before* it filters.

Manual Cardinality Estimates
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
As we have said before, the cardinality is simply the number of rows divided by the number of distinct values (:math:`{\mathit{NDV}}`); a rough estimate of the selectivity is :math:`{1/\mathit{NDV}}`.
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

SQL Optimization Techniques
===========================
Before you start fidgeting with individual SQL statements, it is important to note that hints are probably the last thing you should consider adding when attempting to optimize your code.
There are several levels of optimization and `it is recommended`_ that you start with the server, then the database instance, and finally go down through the database objects to individual statements.
After all, the effect of any changes made to individual statements, particularly hints, may be lost when the database is fine-tuned later on.

As a database developer/architect you may not want to tread the path that leads to the desk of the DBA.
Fortunately, there is a bunch of things you can do to improve the runtime performance of your statements:

* Optimize access structures:

  * Database design and normalization.
  * Tables: heap or index-organized tables, and table or indexed clusters.
  * Indexes.
  * Constraints.
  * Materialized views.
  * Partitioning schemes.
  * Statistics, including a comprehensive refresh strategy.

* Rewrite SQL statements:

  * Exclude projections that are not required.
  * Minimize the amount of work done more than once.
  * Factor subqueries that are used multiple times in the same statement.
  * Use ``EXISTS`` instead of ``IN`` because the former stops processing once it has found a match.
  * Use ``CASE`` and/or ``DECODE`` to avoid having to scan the same rows over and over again, especially for aggregation functions that act on different subsets of the same data.
  * Use analytic functions to do multiple or moving/rolling aggregations with a single pass through the data.
  * Avoid scalar subqueries in the ``SELECT``-list.
  * Use joins instead of subqueries, as it gives the optimizer more room to play around in.
  * Say what you mean and pick the right join: if you only need an inner join don't write an outer join.
  * Add logically superfluous predicates that may still aid in the search for an optimal execution plan, particularly for outer joins.
  * Avoid implicit conversions of data types, especially in the ``WHERE`` clause.
  * Write ``WHERE`` clause predicates with a close eye on the indexes available, including the leading edge of a composite index.
  * Avoid, whenever possible, comparison operators such as ``<>``, ``NOT IN``, ``NOT EXISTS``, and ``LIKE`` without a leading ``'%'`` for indexed columns in predicates.
  * Do not apply functions on indexed columns in the ``WHERE`` clause when there is no corresponding function-based index.
  * Don't abuse ``HAVING`` to filter rows *before* aggregating.
  * Avoid unnecessary sorts, including when ``UNION ALL`` rather than ``UNION`` is applicable.
  * Avoid ``DISTINCT`` unless you have to use it.
  * Use PL/SQL, especially packages with stored procedures (and bind variables) and shared cursors to provide a clean interface through which all data requests are handled.
  * Add hints once you have determined that it is right and necessary to do so.

The advantage of PL/SQL packages to provide all data to users is that there is, when set up properly, exactly one place where a query is written, and that's the only place where you have to go to to change anything, should you ever wish or need to modify the code.
PL/SQL will be in our sights in the next part but suffice to say it is the key to maintainable code on Oracle.
Obviously, ad-hoc queries cannot benefit from packages, but at least they profit from having solid access structures, which are of course important to PL/SQL too.

One important thing to keep in mind is that you should always strive to write efficient, legible code, but that premature optimization is not the way to go.
Premature optimization involves tinkering with access structures and execution plans; it does not include simplifying, refactoring and rewriting queries in ways that enable Oracle to optimally use the database objects involved.

Rewriting queries with or without hints and studying the corresponding execution plans is tedious and best left for `high-impact SQL`_ only: queries that process many rows, have a high number of buffer gets, require many disk reads, consume a lot of memory or CPU time, perform many sorts, and/or are executed frequently.
You can identify such queries from the dynamic performance views.
Whatever you, the database developer, do, be consistent and document your findings, so that all developers on your team may benefit from your experiences.

.. _Oxford Dictionary:  http://www.oxforddictionaries.com/definition/english/hint
.. _instructions: http://allthingsoracle.com/a-beginners-guide-to-optimizer-hints
.. _Jonathan Lewis goes even further: http://jonathanlewis.wordpress.com/2008/05/02/rules-for-hinting
.. _query block: http://jonathanlewis.wordpress.com/2007/06/25/qb_name
.. _global hints: http://www.dba-oracle.com/t_sql_hints_tuning.htm
.. _not embedded in the view itself: http://docs.oracle.com/cd/B19306_01/server.102/b14211/hintsref.htm#i27644
.. _an alphabetical list: http://docs.oracle.com/database/121/SQLRF/sql_elements006.htm#SQLRF51108
.. _Oracle Database 12c Performance Tuning Recipes: http://www.apress.com/9781430261872
.. _diagnose performance issues: http://docs.oracle.com/database/121/ARPLS/d_xplan.htm#ARPLS378
.. _listed in the official documentation: http://docs.oracle.com/database/121/REFRN/refrn10141.htm#REFRN10141
.. _many indexed columns but only few of them are referenced: http://www.apress.com/9781430257585
.. _this blog post: http://jonathanlewis.wordpress.com/2007/02/08/index-combine
.. _starting point: http://tonyhasler.wordpress.com/2008/12/27/bushy-joins
.. _snowstorm schema: http://www.google.com/patents/US20090112793
.. _complex view merging: http://blogs.oracle.com/optimizer/entry/optimizer_transformations_view_merging_part_2
.. _single-column bitmap index on all foreign-key columns of the fact table: http://docs.oracle.com/cd/B19306_01/server.102/b14223/schemas.htm#CIHGHEFB
.. _rewrite: http://docs.oracle.com/database/121/DWHSG/qrbasic.htm#DWHSG018
.. _requires a bitmap index: single-column bitmap index on all foreign key columns of the fact table
.. _minimizes redo generation: http://oracle-base.com/articles/misc/append-hint.php#how-the-append-affects-the-table-size-high-water-mark
.. _a lot of space: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::p11_question_id:1951476814728
.. _drop entire partitions: http://ora600tom.wordpress.com/2012/05/30/append-hint-and-table-space-management
.. _least recently used: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:7828371300346568672
.. _factored subqueries to be materialized: http://www.dba-oracle.com/t_materialize_sql_hint.htm
.. _a global temporary table: http://oracle-base.com/articles/misc/with-clause.php#materialize-hint
.. _based on the block size: http://oracle-base.com/articles/misc/pipelined-table-functions.php#cardinality
.. _extensible optimizer: http://www.oracle-developer.net/display.php?id=427
.. _Tom Kyte's: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:2233040800346569775
.. _Tony Hasler's: http://www.apress.com/9781430259770
.. _versatility: http://www.pythian.com/blog/oracles-opt_estimate-hint-usage-guide
.. _full sample: http://www.oracle-developer.net/display.php?id=427
.. _subquery must be evaluated for each row of the main query: http://blogs.oracle.com/optimizer/entry/basics_of_join_predicate_pushdown_in_oracle
.. _view merging is not possible: http://blogs.oracle.com/optimizer/entry/optimizer_transformations_view_merging_part_1
.. _favour the subquery earlier in the process: http://www.morganslibrary.org/reference/hints.html
.. _individual subqueries rather than all subqueries: http://jonathanlewis.wordpress.com/2007/03/09/push_subq
.. _query block as a parameter: http://www.dba-oracle.com/t_push_subq_hint.htm
.. _global temporary table: http://www.dba-oracle.com/t_precompute_subquery_hint.htm
.. _hard-coded into the main query: http://blog.tanelpoder.com/2009/01/23/multipart-cursor-subexecution-and-precompute_subquery-hint
.. _may not have access to statistics on the remote site(s): http://jonathanlewis.wordpress.com/2013/08/19/distributed-queries-3
.. _Ian Hellström: http://wp.me/p4zRKC-3b
.. _deprecated: http://docs.oracle.com/cd/B12037_01/server.101/b10752/whatsnew.htm
.. _join elimination: http://oracle-base.com/articles/misc/join-elimination.php
.. _a histogram on the join expression: http://oracle-randolf.blogspot.de/2014/05/12c-hybrid-hash-distribution-with-skew.html
.. _All possible values: http://docs.oracle.com/database/121/SQLRF/sql_elements006.htm#BABCJHAF
.. _Expert Oracle SQL: http://www.apress.com/9781430259770
.. _direct path write: http://www.toadworld.com/platforms/oracle/w/wiki/793.direct-path-write.aspx
.. _it is recommended: http://www.dba-oracle.com/art_sql_tune.htm
.. _high-impact SQL: http://www.dba-oracle.com/art_sql_tune.htm
 
.. rubric:: Notes

.. [#dynhints] Even though we have never observed dynamically generated hints in the wild we can still perform a Gedankenexperiment to see why they seem like an odd idea anyway. Suppose you want to provide cardinality estimates with the undocumented ``CARDINALITY`` hint based on the parameter values of a subprogram, for instance the parameter of a table function. You may think this is a great idea because you already know about skew in your data, and you want to provide estimates based on your experience. Fine. Unfortunately, you cannot bind the estimate itself, which means that Oracle requires a hard parse, as the hint is simply a literal. This is tantamount to hard-coding the hint and choosing the statement to run with branches of a conditional statement, which sort of defeats the purpose of generating the estimate dynamically. Creating several alternatives based on parameter values may, however, be useful and beneficial to the performance, especially in cases of severe data skew.

.. [#datamodels] We have no intention of starting a debate on the data model paradigms of Kimball and Inmon. The interested reader will find plenty of insightful articles `on the internet <http://searchbusinessintelligence.techtarget.in/tip/Inmon-vs-Kimball-Which-approach-is-suitable-for-your-data-warehouse>`_.
