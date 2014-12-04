.. _sql-indexes:

*******
Indexes
******* 
Imagine you're feeling nostalgic one day and want to look for a definition in a printed dictionary;
you want to know whether 'crapulent' really is as funny as you think it is.
What makes the dictionary so practical when it comes to finding words and phrases?
Well, entries are listed from A to Z.
In other words, they are ordered.
All you need is the knowledge of the order of the letters in the alphabet.
That's all.
 
It would be very inefficient if you had to read the entire dictionary until you spotted the single item you're interested in.
In the worst case you could end up reading the entire 20 volumes of the Oxford English Dictionary, at which point you probably don't care about the assumed funniness of crapulence any longer.
 
The same goes for databases.
To fetch and retrieve rows in database tables it makes sense to have a simple and fast way to look them up.
That is what an index does.
It is similar to the index in old-fashioned books: the entries are listed in alphabetical order in the appendix, but the page number to which the entries refer does not generally have any structure.
That is, the physical order (when an entry is mentioned in the book) is independent of the logical order (when an entry is listed in the index, from A to Z).
In databases we typically do that with the aid of a `doubly linked list`_ on the index leaf nodes, so that each node refers to its predecessor and is successor; the leaf nodes are stored in a database block or page, which smallest available storage unit in the database.
This data structure makes it easy to run through the list in either direction.
 
The dictionary we mentioned earlier is an example of an index-organized table (IOT) in Oracle parlance;  Microsoft SQL Server calls these objects clustered indexes.
The entire table, or dictionary in our example, is ordered alphabetically.
As you can imagine, index-organized tables can be useful for read-only lookup tables.
For data sets that change frequently, the time needed to insert, update, and/or delete entries can be significant, so that IOTs are generally not recommended.
 
Where an index leaf node is stored is completely independent of its logical position in the index.
Consequently, a database requires a second data structure to sift quickly through the garbled blocks: a balanced search tree, which is also known as a `B-tree`_.
The branch nodes of a B-tree correspond to the largest values of the leaf nodes.
 
When a database does an `index lookup`_, this is what happens:
 
#. The B-tree is traversed from the root node to the branch (or header) nodes to find the pointers to relevant leaf node(s);

#. The leaf node chain is followed to obtain pointers to relevant source rows;

#. The data is retrieved from the table.
 
The first step, the tree traversal, has an upper bound, the index depth.
All that is stored in the branch nodes are pointers to the leaf blocks and the index values stored in the leaf blocks.
Databases can therefore support hundreds of leaves per branch node, making the B-tree traversal very efficient; the index depth is typically not larger than 5.
Steps 2 and 3 may require the database to access many blocks.
These steps can therefore take a considerable amount of time.
 
Oh, and in case you are still wondering: crapulent isn't that `funny at all`_.
 
Developer or Admin?
===================
Indexes can speed up lookups but having too many indexes causes serious performance degradations when inserting, updating, and deleting.
The reason is simple: the database has to maintain the index and the data structures associated with it.
As the index grows, branch and leaf nodes may have to be split, which obviously gobbles up valuable CPU time.
 
Horrible advice you'll sometimes encounter in your life as a database developer is that a DBA is responsible for indexes.
Absolutely not!
The performance of a ``SELECT`` depends on indexes, and the existence of indexes on a table affects ``INSERT``, ``UPDATE``,  and ``DELETE`` statements.
Only a developer knows what queries are typically run, how often a table's data is modified, how it is modified (i.e. single rows or in bulk, normal or direct-path inserts, …) so only a developer can judge whether an index on a particular combination of columns makes sense.
 
Knowledge of indexes is a must for every database developer.
A magnificent reference is Markus Winand's `SQL Performance Explained`_.
If it's not in your library, you're not yet serious about databases!
For the more frugal among us, his `website on indexes`_ is also a valuable resource that covers a lot of what's in the book.
 
Access Paths and Indexes
========================
Let's take another quick look at the access paths we talked about :ref:`earlier <sql-explain-plan>`.
 
The **index unique scan** only returns one row because of a ``UNIQUE`` constraint.
Because of that, Oracle performs only the tree traversal: it goes from the branch node to the relevant leaf node and picks the source row it needs from the table.
 
For an **index range scan** the tree is traversed but Oracle also needs to follow the leaf node chain to find all remaining matching entries. It could be that the next leaf node contains more rows of interest, for instance if you require the maximum value of the current leaf node; because only the maximum value of each leaf block is stored in the branch nodes, it is possible that the current leaf block's maximum index value 'spills over' to the next.
A **table access by index ROWID** often follows an index range scan.
When there are many rows to fetch this combination can become a performance bottleneck, especially when many database blocks are involved.
The cost the optimizer calculates for the table access by index ROWID is strongly influenced by the row count estimates.
 
As the name suggest, a **full index scan** reads the entire index in order.
Similarly, a **fast full index scan** reads the entire index as stored on disk.
It is the counterpart of the **full table access scan**, and it too can benefit from multi-block reads.
 
Statistics
==========
Contrary to what some people may have heard, a *balanced* search tree is, as the name suggests, *always* — read that again, please — always balanced.
It is a myth that you have to rebuild the index whenever the performance of your queries is below par.
There are extremely rare cases when `Oracle recommends`_ that you rebuild the indexes but in almost all cases you do not have to rebuild your indexes. [#rebuild]_


Oracle nowadays automatically collects statistics, so once you create an index, Oracle takes care of the rest.
You can see the schedule and some basic information about the statistics collection with the following statement:
 
.. code-block:: sql
   :linenos:
 
    SELECT
      *
    FROM
      dba_autotask_client
    WHERE
      client_name = 'auto optimizer stats collection'
    ;
 
For most databases the automatic statistics collection is sufficient.
If, however, your database has tables that are being deleted and truncated between collection runs, then it can make sense to go `gather optimizer statistics manually`_.
 
When you create an index, Oracle automatically gathers optimizer statistics because it needs to do a full scan anyway.
As of Oracle Database 12c, the same piggybacking is done for the statistics collection of create-table-as-select (CTAS) and insert-as-select (IAS) operations, which is quite nifty; histograms require additional data scans, so these are not automatically gathered.
The execution plans of CTAS and IAS statements show whether statistics are being collected at runtime: ``OPTIMIZER STATISTICS GATHERING``, right below the ``LOAD AS SELECT`` operation.
 
If you change the definition of an index, you may want to update the statistics.
Please coordinate with the DBA to avoid unwanted side effects, such as degrading the performance of all but your own queries because invalidation of execution plans; gathering statistics does not lock the table, it's like running multiple queries against it. [#invaplan]_
     
Predicates: Equality before Inequality
======================================
An index can be beneficial to your queries' performance when there is some sort of filtering that can be handled efficiently by the index.
The performance is intimately related to the ``WHERE`` clause and the existence (or absence) of indexes on columns specified in the ``WHERE`` clause.
As such, the ``INSERT`` statement is the only one of the unholy insert-update-delete (IUD) trinity that can never benefit from an index: it has no ``WHERE`` clause.
With an ``UPDATE`` or ``DELETE`` you typically have predicates, so they can benefit from fast lookups, even though the maintenance of the index negatively affects the performance; it is a trade-off that has to be evaluated carefully by the developer.
In data warehouse environments it is not uncommon to drop all indexes temporarily and re-create them once the data loaders have completed their business.
Alternatively, you can make your index unusable (i.e. ``ALTER INDEX index_name UNUSABLE``) and once you have imported the data, rebuild it: ``ALTER INDEX index_name REBUILD``.
Only for function-based indexes you can ``DISABLE``  and ``ENABLE`` the index.
 
Predicates show up in execution plans as access, index filter, or table-level filter predicates.
An access predicate corresponds to the start and stop conditions of the leaf node traversal.
During the leaf node traversal index filters can be applied.
In case you filter on a column that is not in the index, the filter predicate is evaluated on the level of the table.
 
It is important that you understand the differences in predicates, because it is critical to the index range scan access method.
Access predicates limit the range of the index to be scanned, whereas index filters are applied to the results of the scanned range.
This is typically the performance bottleneck for index range scans;
if the requested rows are all stored in the same block, then it may not be a problem because Oracle can use a single read operation to fetch the data.
However, you want to keep the scanned index range as small as possible, because the leaf node traversal can take a while.
 
Because index maintenance affects the performance of insert, update, and delete (IUD) statements, it is important that you create enough indexes but no more than needed.
The fewer indexes a table has, the better its IUD performance; the fewer indexes the execution of a query uses, the better its performance.
The question thus is how to optimally define indexes, so that the performance of queries is acceptable while at the same time the performance of IUD statements is not abominable.
       
Primary key and unique constraints automatically generate database indexes.
Sometimes these constraints are sufficient but sometimes you need more.
 
If you only have one column that you ever use to look for data, then that column is your index.
Suppose your application requires an ISBN to return the title, the first and last name of the main author, purely as information and nothing else.
It can then make sense to include these three columns in the index too, not because you filter on them — because we just said you don't — but to make sure that Oracle can avoid a trip to the table: Oracle can simply read the index and retrieve the information it needs to provide from it.
Such retrieval logic is generally coded in PL/SQL functions that live inside the database.
We shall talk more about functions when we talk about PL/SQL, but we want to mention two advantages of this approach to whet your appetite: when using bind variables, the execution plan is stored irrespective of the ISBN provided, so Oracle does not need to ask the optimizer again and again, and you can take advantage of the result cache, which means that information on popular books is cached and available without having to check the index or fetch data from the table.
 
Anyway, when you include multiple columns in an index, that is you create a compound index, the order of the columns is very important.
The difference between a column being used as a filter rather than access predicate can be significant.
Because of that, it is recommended to index for `equality first`_ and then for ranges, typically ``DATE``-like columns, so that the extent of the leaf node traversal can be kept to a minimum.
 
Let's take another look at our :ref:`friendly household robot <sql-proc-order>`.
We wanted beer chilled below five degrees.
The time we have to wait for the robot to fetch the items from the fridge depends on how fast our electronic companion can locate the items requested.
An index is of course beneficial but the question remains what we want to index first: temperature or product.
Because we rarely ever need an item at an exact temperature, that is we want items in a particular temperature range, it makes sense to index on product first and then on temperature.

'What's the difference?' you ask?
Well, let's find out.

.. _fig-index-range-eq:

.. figure:: images/index-nodes-range-eq.*
   :scale: 60%
   :alt: Index branch and leaf nodes for range (temperature) and then equality (product)
 
When the index is on temperature first and then product, the robot checks one of the branch nodes and sees that there are *at least* two leaf nodes with the correct temperature; the temperature is the access predicate.
It follows the pointers to these leaf nodes and uses the product column as a filter predicate.
Because one of the leaf nodes' maximum value for temperature is 4, we have to follow the leaf node chain because the next leaf node may contain more products with a temperature below five degrees.
And yes, there is one more item; it's not what we were looking for but our robot could not know that when browsing the index.

.. _fig-index-eq-range:

.. figure:: images/index-nodes-eq-range.*
   :scale: 60%
   :alt: Index branch and leaf nodes for equality (product) and then range (temperature)
 
Assume the product is index first and the temperature second, as recommended.
What's the advantage?
Well, the index tells us exactly in which leaf nodes to look for beer: it's in the ones before cream because ``'apple' < 'beer' < 'cream'``.
We still have the case that we have to follow the leaf node chain because ``'beer'`` happens to be the last product in one of the index leaf nodes.
 
Note that the advice about equality-range indexes is not the same as saying that the most selective index in our case should go first.
Suppose, for the sake of argument, that each product in our table has a different temperature.
Depending on the temperature of the warmest item, we could have to scan the entire table, if all products have been in the fridge at least a few hours, so that they are all below five degrees, or we could have no items to consider at all because someone forgot to close the fridge door and everything is almost at room temperature.
Sure, in the latter case an index on the temperature first would yield the best performance.
But: in the former case the performance could potentially be horrible, especially if the number of products in the fridge were to increase significantly.
 
Another scenario to ponder about: all products in the fridge are unique.
For the product-temperature index we would look up the product and then verify whether its temperature is in the specified range.
Simple. Quick.
For the temperature-product index, however, we could potentially have to scan everything and then filter out the items that are not beer.
 
Yet another scenario: all products in the fridge are beer — man, you're thirsty!
The product-temperature index requires us to do a full fridge scan and take only the bottles below five degrees.
The temperature-product index is obviously more efficient because it can use the temperature as an access predicate; the filter predicate on ``'beer'`` is pretty much useless, as is the index.
 
As you can see, the performance of an equality-range index is more consistent and thus more production-suitable than the range-equality index.
There are of course cases when the range-equality index is more appropriate: skewed data, where you *know* it is more advantageous, but you have to be absolutely sure the skewness stays the same.
For most databases that is a bit too iffy to be useful advice.
 
Another reason why the equality-range index is a good rule of thumb is that whatever is searched for with an equality predicate is something that is pretty much standard to all your queries: you primarily want certain stuff from the fridge, where the temperature is only secondary.
"I'm starving for some 7-degrees' produce," is not something you often hear people say when they're thirsty but have no cool liquids available; you might want to gobble up some cool broccoli instead but I doubt it.
 
If our fridge table is equipped with expiration dates, that column would also be included as a second or third column.
We're typically interested in items that have not yet expired (``expiration_date <= SYSDATE``), or, if we want to have the robot clean up the fridge, all items that have already expired.
Whether the temperature or expiration date should go first *after* the product depends a bit on the situation: do you search more frequently for the expiration date or the temperature of items in the fridge?
 
Anyway, when you need an index on additional columns, add these to the index you already have or redefine it.
An extra index may not provide you with the benefits you expect: the optimizer has to combine two indexes when executing your queries, and the database has to maintain two indexes.
The fewer indexes the optimizer has to use, the better the performance of your queries.
More than 5 indexes is usually not recommended, but the exact number may well depend on the specifics of your environment.
Nevertheless, if you are really looking at five or more indexes for a particular table, you have to think about why you need so many separate indexes, and document your reasons carefully.
 
With regard to SQL statements, always be as specific as possible.
Suppose you go to the trouble of adding manufacturers of products in your fridge, you create a compound manufacturer-product index, and let the legged circuit board look for some ``'Coke Zero'`` by ``'The Coca-Cola Company'``.
Sure, ``'Coke Zero'`` is only made by one company, but today you're too tired, so you simply write ``WHERE product = 'Coke Zero'``.
If you're lucky, the robot decides to do a skip scan on the leading edge of the index; if you are less fortunate, and your fortune depends mainly on the histogram of the leading index column (i.e. the manufacturer column) , your robot may decide on a full fridge scan.
Oracle does not know about correlations in your data, so if you want Oracle to come back with your rows as quickly as possible, provide all the details possible that aid it in its search for your data.
If at all possible, always include your leading index column in all your queries' predicates.
It is advice given to mathematicians and, likewise, applies to (database) developers: do not assume anything unless stated otherwise.

Predicates: LHS vs RHS
======================
They way you *syntactically* write predicates matters, even though *logically* various forms of predicates are equal.
The difference between the left-hand side (LHS) and the right-hand side (RHS) of equality and inequality predicates is significant.
Oracle only evaluates the right-hand side of predicates for the compilation, and the left-hand side should ideally refer to indexed columns as they appear in the index.
 
What about predicates that emulate full-text searches like ``WHERE col_name LIKE '%something interesting%'``?
Short answer: you're pretty much screwed.
Standard indexes are not designed to meet that requirement.
It's like asking you to search for a book with an ISBN that has 123 somewhere in it.
Good luck!
Long answer: `Oracle Text`_.
Yes, it's the long answer, even though it's only two words, because it requires you to do some digging.
Oracle Text comes with all editions of the database but it's beyond our scope.
With it you can use SQL to do full-text searches, which is especially interesting you need to mine texts; it's overkill for occasional queries with a non-leading ``LIKE`` though.
 
Function-Based Indexes and NULLs
================================
By default Oracle does not store null rows in a (B-tree) index.
You can add them with a simple trick:
 
.. code-block:: sql
   :linenos:
 
   CREATE INDEX index_name
     ON tab_name ( nullable_col_name, 1 );
 
The 'trick' is of course nothing but a function-based index.
By adding nulls to your (function-based) index, you ensure that Oracle is able to avoid full table scans when you ask for ``col_name IS NULL`` .
Alternatively, you can use ``NVL`` as a function in your index if you want to; you have to remember that your index can only be used if you use the same function in your filter predicates.
 
That is a common thread in function-based indexes though: you have to have the exact same predicate as used in the index for the index to be used.
Oracle has no compiler that evaluates and simplifies (mathematical) expressions, so a ``WHERE`` clause like ``WHERE col_a + col_b = 42`` does not use an index on ``col_a`` because the lef-hand side also includes ``col_b``.
To use an index on ``col_a`` , you have to rewrite the predicate as ``WHERE col_a = 42 - col_b``.
Similarly, ``LN ( EXP(col_real) )`` is not simplified to ``col_real`` for ``col_real`` a column of real-valued numbers.
Oracle is smart but you cannot expect it to do everything for you: not even state-of-the-art computer algebra systems like Mathematica and Maple can simplify all crazy expressions you can think of.
 
The power of function-based indexes lies in the fact that often your applications have to filter for bits and pieces of data that are already available but normal indexes cannot be used, which often happens because of conversion, mathematical, and string-manipulation functions, in particular ``SUBSTR()`` and  ``LOWER()`` or ``UPPER()``.
Suppose you have a sudden, inexplicable urge to behave like a business analyst and you want to generate a report of the average temperature of all products with an expiration date of products in your fridge for a particular ISO workweek; if you think this is an odd request then please replace temperature with amount, expiration date with the date of the transaction, and the fridge with a sales table.

You create the following function-based index: ``CREATE INDEX ix_workweek ON fridge ( TO_CHAR(expiration_date, 'IW') )``.
If you now use a clause like ``WHERE TO_CHAR(expiration_date, 'IW') = '20'``, you can see all products with an expiration date in workweek twenty using the index ``ix_workweek``; the single quotes in the ``WHERE`` clause are included because the resulting expression is of type ``VARCHAR2``.
Avoid implicit conversions as much as possible; not because of the almost negligible conversion performance penalty but because you rely on Oracle to do it right in the background *and* it is considered bad style!
 
Imagine you have created a function-based index on a certain concatenation of columns, for instance ``manufacturer || '''s ' || product``, then you can use that exact expression in the ``WHERE`` clause.
This is, however, an extremely fragile and overly complex solution that does not really belong in your queries.
Such logic is usually application-specific, so it should either be handled by the application itself or a layer of PL/SQL between the database and your user interface that extracts the data and then with the aid of an auxiliary function formats it correctly: it is always a good idea to separate the data-access layer from the application layer, as it minimizes the number of places you have to search and replace something whenever a change requests ends up on your desk.
 
Why?
What if next week someone decides to search for ``WHERE product || ' by '  || manufacturer = ...`` instead?
You then need to change not only your query but also your index.
Worse still, what if you want to list only stuff from one manufacturer?
You can't even use the index!
Why make life hard when you can just add both ``manufacturer`` and ``product`` to your index and search for each one individually, separated by a beautiful ``AND``?!
If at this point you think that no one is that thick, then I'll just say that if I'd have had a dollar for each time I saw something similar (usually with first and last names), I'd be rich.
And if I'd have had an extra dollar for each time people complained about shitty performance because of such an abomination of a predicate and demanded an index to solve it, I'd be stinking rich.
 
By the way, we're not done with nulls yet.
Queries can sometimes run `without utilizing an index`_ because a ``NOT NULL`` constraint is absent.
Constrains are thus not only important to enforce consistency but also to ensure consistent performance of your queries.
Furthermore, functions on columns *with* ``NOT NULL`` constraints can lead to the same (unwanted) behaviour.
The reason is that Oracle does not know whether a function preserves the ``NOT NULL`` constraint of the column.
For some internal functions, though, Oracle knows that ``NOT NULL`` is preserved, which means that it can still use any available and relevant indexes.
Examples of such internal functions are ``LOWER()`` and ``UPPER()``.
 
User-defined functions are black boxes as far as the optimizer is concerned.
As of Oracle Database 11g, `virtual columns`_ can be used to circumvent the issue.
Virtual columns are not stored and they are derived (or computed) from other columns in the same table.
They are created like normal columns but with the syntax of ``col_name [ data_type ] [ GENERATED ALWAYS ] AS ( expression ) [ VIRTUAL ]``, where the entries between square brackets are optional although highly recommended to indicate that the column in question is merely virtual.
An index on a virtual column is like a function-based index on a normal column, but it has the benefit that you can add the ``NOT NULL`` constraint to it.
Hence, the optimizer can treat the expression as a ``NOT NULL``-preserving function.
Sweet!

Predicates: The ``WHERE`` Clause
================================
The ``WHERE`` clause is the one that determines whether or not indexes can be used efficiently.
One side of each predicate must be as specified in the index(es) for Oracle to be able to use any index.
Whether it is the left-hand side or the right-hand side is irrelevant, although typically it is the left-hand side because SQL is written from the left to the right.
Note that the order sometimes matters though: ``col_name LIKE 'ABC%'`` is not the same as ``'ABC%' LIKE col_name``.
The former searches for ``col_name`` entries that begin with ``ABC``, whereas the latter is the same as the filter ``col_name = 'ABC%'``, that is the ``%`` is not interpreted as a wild card at all.
 
Indexes can only be used when predicates are sargable, or search-argument-able, which admittedly is a horrible phrase.
Functions on columns in the index can prohibit index use, particularly when the index is not a function-based index.
Apart from that, some operators are sargable and optimizable (i.e. allow the use of an index): ``=``, ``<``, ``>``, ``>=`` ``IS NULL``; some operators are sargable yet not optimizable: ``<>`` and its equivalents (i.e. ``!=`` and ``^=``) and ``NOT``; and ``LIKE`` with a leading wild card is not sargable and hence not optimizable.
Sargable predicates can be pushed down, which means that a predicate in a statement that references a view or derived table can be 'pushed down' to the view or derived table itself, which avoids having to scan the entire underlying data structure only to filter out a few relevant rows later.
Sargable, non-optimizable predicates can still benefit from the optimizer's efforts; non-sargable predicates cannot though.
 
A SQL statement that links several sargable predicates with an ``OR`` cannot be optimized when the predicates involve different columns.
If, however, the predicates can be rewritten as an equivalent ``IN``-list, which Oracle does internally as a part of its predicate transformations, then Oracle can indeed optimize the statement and therefore utilize existing indexes.
 
Important is, as always, that the data type of each search term matches the data type of the indexed column or expression; it is best that you convert search terms on the right-hand side if necessary but leave the left-hand side as is.
Unnecessary use of ``TO_CHAR()`` and ``TO_NUMBER()`` (on the left-hand side) is not only sloppy but it can hamper index use.
The same goes for ``NVL()`` and the like.
 
If you often encounter fixed expressions or formulas in your predicates, you can create function-based indexes on these expressions.
Make sure that the columns referenced appear in the index in exactly the same way as they appear in the predicates, *and* make sure that the right-hand side does not contain the columns from the index: Oracle does not solve your equations for you.
 
Predicates that are often badly coded include operations on dates.
Yes, it is possible to create a function-based index on ``TRUNC ( expiration_date )`` and use same expression in the database.
However, *all* predicates on the column ``expiration_date`` *must* include ``TRUNC()`` for Oracle to be able to use the index in all cases.
A simple and elegant solution is to provide ranges, either with ``>= TO_DATE(...)`` and ``<= TO_DATE(...)`` or with ``BETWEEN TO_DATE(...) AND TO_DATE``, which is inclusive.
Should you not want it to be inclusive subtract a minimal interval like so: ``TO_DATE(...) - INTERVAL '1' SECOND'``.

Why not the literal ``1/86400`` or ``1/(24*60*60)``?
Well, it may be easy for you to understand something like that because you wrote it (and perhaps added a comment), but it is not always easy to fathom such literals, especially if developers simplify their fractions as in ``7/10800``, which is 56 seconds by the way.
The index may not care about how you write your literals but the other developers in the team do care.
Let the code speak for itself!
 
Since we're on the topic of dates: *never* write ``TO_CHAR ( expiration_date, 'YYYY-MM-DD' ) = '2014-01-01'``.
Leave the ``DATE`` column as is and write ``expiration_date >= TO_DATE ( '2014-01-01','YYYY-MM-DD' )`` and ``expiration_date < TO_DATE ( '2014-01-01','YYYY-MM-DD' ) + INTERVAL '1' DAY`` instead. [#interval]_
Yes, it's a bit more typing, but that way an index range scan can be performed and you do not need a function-based index.
 
'But what if I need only products from the fridge that expire in February?'
Since repetition is the mother of learning, here comes: specify ranges from the first day of February to the last day of February.
 
'But I want to show the total number of products by the year and month of the expiration date.'
You could use the ``EXTRACT ( YEAR FROM expiration_date )`` and similarly for the month, ``TRUNC( expiration_date, 'MM' )`` or ``TO_CHAR ( expiration_date, 'YYYY-MM' )``.
However, since you are pulling in all data from the table, a full table scan really is your best option.
Yes, you read that right: a full table scan is the best alternative; we'll say more about full table scans in a few moments.
Furthermore, if you already have an index on ``expiration_date`` and it is stored in order (i.e. it is not a ``HASH`` index on a partitioned table), then the ``GROUP BY`` can make use of the index without any additional function-based indexes.
 
The ``LIKE`` comparison operator is also often a cause for performance problems because applications tend to allow wild cards in strings, which means that a search condition à la ``WHERE col_name LIKE '%SOMETHING%'`` is not uncommon.
Obviously, you cannot create a sensible index for a predicate like that.
It is tantamount to asking a dictionary to provide you with a list of all possible sequences of characters in any position.
 
The ``INDEX`` hint, as described by `Laurent Schneider`_, is — contrary to what is claimed by the said author — *not* always beneficial for predicates with leading and trailing wild cards, so be sure to try it out.
An index is, however, used when such a predicate is specified with bind variables:
 
.. code-block:: sql
   :linenos:
 
    VARIABLE l_like VARCHAR2(20);
    EXEC :l_like := '%SOMETHING%';
 
    SELECT
      *
    FROM
      tab_name
    WHERE
      col_name LIKE :l_like;
 
If you always look for things *ending* with a series of characters, such as ``LIKE '%ABC'`` you *can* use an index.
Just create the index on ``REVERSE ( col_name )`` and reverse the string you are looking for itself, and voilà, it works: ``WHERE REVERSE ( col_name ) LIKE 'CBA%'``.
 
To search in a case-insensitive manner you have to create a function-based index, say, ``UPPER(col_name)``.
You could have gone with ``LOWER(col_name)`` and whatever you prefer is really up to you.
All that matters is that you are thrifty and consistent: switching back and forth between ``UPPER()`` and ``LOWER()`` is a bad idea because the database has to maintain two indexes instead of one, and you really only need one index.
Which function you choose for case-insensitive searches is irrelevant but document whichever you choose, so it is clear to all developers on the team.
 
In an international setting you may want to use ``NLS_UPPER( col_name, 'NLS_SORT = ...' )``, so that for instance — for ``... = XGERMAN`` — ``ß`` and ``ss`` are seen as equivalent.
The parameters ``NLS_SORT`` and ``NLS_COMP`` can be made case- or accent-insensitive by appending ``_CI`` or ``_AI`` to their `sort name values`_ respectively.
The ``NLS_SORT`` parameter can be used to alter a session or the entire system.
 
For purely linguistic rather than binary searches of text, you can set the system's or session's ``NLS_COMP = LINGUISTIC``.
The performance of linguistic indexes can thus be improved: ``CREATE INDEX ix_col_name_ling on tab_name ( NLSSORT( col_name, 'NLS_SORT = FRENCH' ) )``, for French for example.
 
We have already seen that with function-based indexes it is important to have the exact same expression save for irrelevant spaces.
A functionally equivalent expression that is syntactically different prohibits the use of an index, so writing ``REGEXP_LIKE()`` in your ``WHERE`` clause when you have used ``INSTR()`` in the index means that the optimizer will ignore the index.
 
For Oracle Database 11g there is a good book on `expert indexing`_, if you want to learn more about indexes.

Full Table Scans
================
Full table scans are often seen as a database's last resort: you only do them if you absolutely have to.
That reputation of full table scans is not entirely warranted though.
 
For small tables it often does not make sense for Oracle to read the associated index, search for the relevant ROWIDs, and then fetch the data from the database tables when it can just as easily do a single round trip to the table.
Thanks to multiblock I/O in full table scans a couple of parallel round trips are also possible to speed up the process.
 
Analogously, when the database has to return a sizeable portion of all the rows from a table, the index lookup is an overhead that does not always pay off.
It can even make the database jump back and forth between blocks.
 
Full table scans frequently indicate that there is optimization potential but remember, as originally noted by `Tom Kyte`_: "full table scans are not evil, indexes are not good".
 
Top-N Queries and Pagination
============================
Top-N and pagination queries frequently pop up in applications: a user is only shown the top-N entries or allowed to flip back and forth between pages of data.
Prior to Oracle Database 12c there were a couple of `roundabout methods`_ available to do pagination: `offset, seek`_, `window or analytical functions`_.
 
The ``OFFSET/FETCH`` or `row-limiting clause`_ has greatly simplified life for developers:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 10,11
 
   SELECT
      manufacturer
    , product
    , temperature
    , expiration_date
   FROM
      fridge
   ORDER BY
      expiration_date
   OFFSET 5 ROWS
   FETCH NEXT 10 [ PERCENT ] ROWS ONLY  
   ;

An issue that is often overlooked when it comes to the row-limiting clause is explained on `Markus Winand's Use The Index, Luke`_ page.
We'll briefly cover the salient details, as it affects application and database performance.
Suppose your users flip through pages of data and are allowed to insert rows at any position.
The ``OFFSET`` clause can cause rows to show up twice: once on the previous page *before* the row was inserted and once on the current page *after* the row was inserted (on the previous page).
Furthermore, ``OFFSET`` is implemented in a way that data below the ``OFFSET`` line needs to be fetched and sorted anyway.
 
The solution to this conundrum is quite simple: keyset pagination: use the ``FETCH`` clause as before but replace the ``OFFSET`` clause with a ``WHERE`` clause that limits the result set to all rows whose key is before or after the identifier (key) of the row previously displayed.
Whether you have to take ``>`` or ``<`` depends on how you sort and what direction the pagination runs in of course.
An index on the columns in your ``WHERE`` clause, including the key, to aid the ``ORDER BY`` means that browsing back to previous pages does not slow your users down.
 
With that in mind we can rewrite our query:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 9,12

   SELECT
      manufacturer
    , product
    , temperature
    , expiration_date
   FROM
      fridge
   WHERE
      expiration_date < last_expiration_date_of_previous_page
   ORDER BY
      expiration_date
   FETCH NEXT 10 [ PERCENT ] ROWS ONLY
   ;
 
Two major bummers of keyset pagination are that 1) you cannot jump to arbitrary pages because you need the values from the previous page and 2) no convenient bidirectional navigation is available because that would require you to reverse the ``ORDER BY`` and key comparison operator.
 
Index-Organized Tables
======================
Index-organized tables are generally narrow lookup tables.
They have to be narrow because all columns of the table are in the index.
In fact, the index is the table itself.
 
It is technically possible to add additional indexes to an index-organized table.
However, accessing an index-organized table via a secondary index is very inefficient.
The reason is that the secondary index cannot have pointers to rows in the table because that would require the data to stay where it is.
Forever.
Because the data is organized by the primary index in an index-organized table, it can move around whenever data in modified.
Secondary indexes store logical instead of physical ROWIDs; `logical ROWIDs`_ (``UROWID``) contain physical guesses, which identify the block of the row at the time when the secondary index was created or rebuilt.
Standard heap tables are generally best for tables that require multiple indexes.
 
Index-organized tables can be beneficial to OLTP applications where fast primary-key access is essential; inserts typically take longer for index-organized tables.
Because the table is sorted by the primary key, duplication of data structures is avoided, reducing storage requirements.
Key compression, which breaks an index key into a prefix and suffix, of which the former can be shared among the suffix entries within an index block, reduces disk storage requirements even further.
 
Index-organized tables cannot contain virtual columns.
 
Beyond B-Trees: Bitmap Indexes
==============================
For columns with low cardinality the classical B-tree index is not an optimal solution, at least not in DSS or OLAP environments.
Bitmap indexes to the rescue!
 
Bitmap indexes use compression techniques, which means that many ROWIDs can be generated with very little I/O.
As argued by `Vivek Sharma`_ and `Richard Foote`_, a bitmap index is not only your go-to index for low-cardinality columns but also for any data that does not change frequently, for instance fact tables in data warehouses.
Nevertheless, concurrent IUD operations clearly tip the scales in favour of standard B-tree indexes; bitmap indexes are problematic for online applications with many concurrent DML statements because of deadlocks and the overhead to maintain bitmap indexes.
 
Ad hoc queries are also generally handled better by bitmap than B-tree indexes.
Queries with ``AND`` and ``OR`` can be executed efficiently because bitmap indexes on non-selective columns can be combined easily; ``COUNT`` queries are handled particularly efficiently by bitmap indexes.
If users query many different combinations of columns on a particular table, a B-tree index has no real candidate for the leading index column.
A bitmap index on all columns typically queried by analysts allows the index to be used for all these queries.
It does not matter whether your business users use only one, two, three, or all columns from the index in their queries.
In addition, nulls are included in the bitmap index, so you don't have to resort to function-based indexes.
 
By the way, you *can* create `bitmap indexes on index-organized tables`_.
More information on default B-tree and other indexes is of course `provided by Oracle`_.
 
.. _doubly linked list: http://en.wikipedia.org/wiki/Doubly_linked_list
.. _index lookup: http://www.orafaq.com/node/1403
.. _B-tree: http://use-the-index-luke.com/sql/anatomy/the-tree
.. _funny at all: http://www.oxforddictionaries.com/definition/english/crapulent
.. _SQL Performance Explained: http://sql-performance-explained.com
.. _website on indexes: http://use-the-index-luke.com
.. _Oracle recommends: https://blogs.oracle.com/sysdba/entry/when_to_rebuild_index
.. _gather optimizer statistics manually: http://docs.oracle.com/database/121/TGSQL/tgsql_stats.htm#TGSQL415
.. _equality first: http://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates
.. _Oracle Text: http://www.oracle.com/technetwork/database/enterprise-edition/index-098492.html
.. _Tom Kyte: https://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:9422487749968
.. _without utilizing an index: http://use-the-index-luke.com/sql/where-clause/null/not-null-constraint
.. _virtual columns: http://www.oracle-base.com/articles/11g/virtual-columns-11gr1.php
.. _offset, seek: http://use-the-index-luke.com/sql/partial-results/fetch-next-page
.. _roundabout methods: http://www.oracle-base.com/articles/12c/row-limiting-clause-for-top-n-queries-12cr1.php
.. _window or analytical functions: http://use-the-index-luke.com/sql/partial-results/window-functions
.. _row-limiting clause: http://docs.oracle.com/database/121/SQLRF/statements_10002.htm#SQLRF55631
.. _logical ROWIDs: http://docs.oracle.com/database/121/CNCPT/indexiot.htm#CNCPT911
.. _Markus Winand's Use The Index, Luke: http://use-the-index-luke.com/no-offset
.. _Vivek Sharma: http://www.oracle.com/technetwork/articles/sharma-indexes-093638.html
.. _Richard Foote: http://richardfoote.wordpress.com/2010/02/18/myth-bitmap-indexes-with-high-distinct-columns-blow-out
.. _bitmap indexes on index-organized tables: http://docs.oracle.com/database/121/ADMIN/tables.htm#ADMIN11699
.. _provided by Oracle: http://docs.oracle.com/database/121/ADMIN/indexes.htm#ADMIN11709
.. _Laurent Schneider: http://laurentschneider.com/wordpress/2009/07/how-to-tune-where-name-likebc.html
.. _sort name values: http://docs.oracle.com/database/121/NLSPG/applocaledata.htm#NLSPG593
.. _expert indexing: http://www.apress.com/9781430237358
 
.. rubric:: Notes

.. [#invaplan] The ``DBMS_STATS.AUTO_INVALIDATE`` option can be used to ensure that Oracle does not invalidate all cursors immediately, which can cause a significant CPU spike. Instead, Oracle uses a rolling cursor invalidation based on internal heuristics.

.. [#rebuild] The index clustering factor indicates the correlation between the index order and the table order; the optimizer takes the clustering factor into account for the ``TABLE ACCESS BY INDEX ROWID`` operation. A high ratio of leaf nodes marked for deletion to leaf nodes (> 0.20), a low value of percentage used (< 0.80), and a clustering factor (see ``DBA_INDEXES``) close to the number of rows (instead of the number of blocks) in the table (see ``DBA_SEGMENTS``) are indicators that your indexes may benefit from rebuilding. If the clustering index is close to the number of rows, then the rows are ordered randomly.

.. [#interval] The ``INTERVAL`` function has one major disadvantage: ``SELECT TO_DATE ( '2014-01-31', 'YYYY-MM-DD' ) + INTERVAL '1' MONTH FROM dual`` leads ``ORA-01839: date not valid for month specified`` error. The function ``ADD_MONTHS()`` solves that problem.
