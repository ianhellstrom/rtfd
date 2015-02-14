.. _plsql-loops:

***********************************
Loops, Cursors, and Bulk Operations
***********************************
Oracle generously provides a list of things developers can do to `tune their PL/SQL code`_.
One item from that list is probably the single best tool developers have at their disposal to supercharge their code: bulk processing.
 
Whenever you need to retrieve and modify more than one row of a database table you have a few options:
 
* A single SQL statement.
* A cursor ``FOR`` loop to iterate through the results one row at a time to insert, update and/or delete records.
* Retrieve and temporarily store all records in a collection (``BULK COLLECT``), and process the rows
 
  * one at a time with a ``FOR`` loop;
  * in bulk with ``FORALL``.
 
* Retrieve all records in batches with an explicit cursor and store the results in a collection (``BULK COLLECT ... LIMIT``), after which the rows are processed
 
  * one at a time with a ``FOR`` loop;
  * in bulk with ``FORALL``.
 
Even though it is possible to use a single SQL statement to modify data in a table, PL/SQL developers rarely take this approach.
One of the primary reasons against such a single statement is that it is an all-or-nothing proposition: if anything goes wrong all modifications are rolled back.
With PL/SQL you have more control over what happens when exceptions occur.
 
The advantage of ``FORALL`` is that the statements are sent (in batches) from PL/SQL to the SQL engine `in one go`_, thereby minimizing the number of :term:`context switches <context switch>`.
A disadvantage is that with ``FORALL`` you can only throw one DML statement over the fence to the SQL engine, and the only differences allowed are in the ``VALUES`` and ``WHERE`` clauses of the modification statement.
Beware that when multiple ``INSERT`` statements are sent to the SQL statement executor with ``FORALL``, any statement-level triggers will fire only once: either before all statements have been executed or after all statements have completed.
 
``BULK COLLECT`` does for queries what ``FORALL`` does for :term:`IUD statements`; ``MERGE`` is supported from Oracle Database 11g onwards.
Since `collections`_ are required in ``BULK COLLECT`` and ``FORALL`` statements, and (pipelined) table functions, where they can greatly improve the runtime performance, we take a moment to go through the fundamentals.
 
Collections
===========
PL/SQL has three *homogeneous* one-dimensional collection types: associative arrays (PL/SQL or index-by tables), nested tables, and variable-size or varying arrays (varrays).
Homogeneous refers to the fact that the data elements in a collection all have the same data type.
Collections may be nested to simulate multi-dimensional data structures, but these are currently not supported with the traditional syntax you may be familiar with from other programming languages, such as C# or Java.
 
Summarized in the table below are the distinguishing features of each of these three collection types, where we have omitted ``[ CREATE [ OR REPLACE ] ] TYPE type_identifier IS ...`` from the declarations:
 
+---------------------+-------------------------------------------------+------------------+--------+-------+------------+----------------+------------+----------+---------------+
| Collection          | Type declaration                                | Index            | Sparse | Dense | Persistent | Initialization | ``EXTEND`` | ``TRIM`` |  ``MULTISET`` |
+=====================+=================================================+==================+========+=======+============+================+============+==========+===============+
| Nested table        | ``TABLE OF data_type``                          | positive integer | Yes*   | Yes   | Yes        | Yes            | Yes        | Yes      | Yes           |
+---------------------+-------------------------------------------------+------------------+--------+-------+------------+----------------+------------+----------+---------------+
| Associative array   | ``TABLE OF data_type INDEX BY ix_data_type``    | ``ix_data_type`` | Yes    | Yes   | No         | No             | No         | No       | No            |
+---------------------+-------------------------------------------------+------------------+--------+-------+------------+----------------+------------+----------+---------------+
| Variable-size array | ``VARRAY( max_num_elems ) OF data_type``        | positive integer | No     | Yes   | Yes        | Yes            | Yes        | Yes      | No            |
+---------------------+-------------------------------------------------+------------------+--------+-------+------------+----------------+------------+----------+---------------+
 
Here, ``ix_data_type`` can be either a ``BINARY_INTEGER``, any of its subtypes, or a ``VARCHAR2``.
Associative arrays are thus the only collection type that can handle *negative* and *non-integer* index values.
Please note that when it comes to performance, the difference between integers and small strings (i.e. fewer than 100 characters) as indexes is minimal; for large strings the overhead of hashing can be quite significant, as demonstrated by `Steven Feuerstein and Bill Pribyl`_.
 
We have added an asterisk to the 'Yes' in the column 'Sparse' for nested tables because technically they can be sparse, although in practice they are often dense;
they only become sparse when elements in the middle are deleted after they have been inserted.
The only collection type that can be used in PL/SQL blocks but neither in SQL statements nor as the data type of database columns is an associative array.
Although both nested tables and varrays can be stored in database columns, they are stored differently.
Nested table columns are stored in a separate table and are intended for 'large' collections, whereas varray columns are stored in the same table and thought to be best at handling 'small' arrays.
 
Nested tables and variable-size arrays require initialization with the default constructor function (i.e. with the same identifier as the type), with or without arguments.
All collections support the ``DELETE`` method to remove all or specific elements; the latter only applies to nested tables and associative arrays though.
You cannot ``DELETE`` non-leading or non-trailing elements from a varray, as that would make it sparse.
 
As you can see, the ``TRIM`` method is only available to nested tables and varrays; it can only be used to remove elements from the back of the collection.
Notwithstanding this restriction, associative arrays are by far the most common PL/SQL collection type, followed by nested tables.
Variable-size arrays are fairly rare because they require the developer to know in advance the maximum number of elements.
 
Note that Oracle does not recommend using ``DELETE`` and ``TRIM`` on the same collection, as the results may be `counter-intuitive`_: ``DELETE`` removes an element but retains its placeholder, whereas ``TRIM`` removes the placeholder too.
Running ``TRIM`` on a previously deleted element causes a deleted element to be deleted.
Again.
 
The built-in ``DBMS_SQL`` package contains a couple of `collection shortcuts`_, for instance: ``NUMBER_TABLE`` and ``VARCHAR2_TABLE``.
These are nothing but associative arrays indexed by a ``BINARY_INTEGER`` based on the respective data types.
 
To iterate through a collection you have two options:
 
* A numeric ``FOR`` loop, which is appropriate for *dense* collections when the entire collection needs to be scanned: ``FOR ix IN l_coll.FIRST .. l_coll.LAST LOOP ...``.
* A ``WHILE`` loop, which is appropriate for sparse collections or when there is a termination condition based on the collection's elements: ``WHILE ( l_coll IS NOT NULL ) LOOP ...`` .
 
When using the ``FORALL`` statement on a sparse collection, the ``INDICES OF`` or ``VALUES OF`` option of the `bounds clause`_ may prove equally useful.
 
Now that we have covered the basics of collections, let's go back to the performance benefits of bulk processing.
 
Performance Comparisons
=======================
There are quite a few performance comparisons documented on the internet and in books.
We shall not provide our own as that would be mainly a repetition and thus a waste of time.
Instead, we try and bring together all the information, so that you, the database developer, can benefit from what others have already figured out.
Our service is in gathering the information, so that you don't have to wade through all the sources yourself.
 
Explicit vs Implicit Cursors
----------------------------
The discussion on whether explicit cursors (i.e. with ``OPEN``-``FETCH``-``CLOSE``) are always to be preferred to implicit ones stems from an era that has been `rendered obsolete`_ by Oracle.
The performance of explicit cursors in all but prehistoric Oracle versions is more or less on par with that of implicit cursors.
In fact, sometimes an implicit cursor `can be faster`_ than an explicit one, even though it does more work behind the scenes.
 
Apart from that, a developer should *always* be wary of experts claiming that A is always to be preferred to B, especially when that advice is based on comparisons done on previous versions --- yes, we are fully aware of the fact that our advice is reminiscent of the `liar's paradox`_.
Companies like Oracle continue to invest in their products, and features that were once considered slower but more convenient are often improved upon to make them at least as fast as the traditional approaches.
 
We pause to remark that the ``LIMIT`` clause is part of the ``FETCH`` statement and thus only available to explicit cursors.
 
The Impact of Context Switches
------------------------------
'Are context switches really such a big deal?'
 
We could argue that adding little bits of overhead to each DML statement inside a cursor ``FOR`` loop, which --- as we have seen just now --- can be based on either an explicit or implicit cursor, that iterates over a large data set can quickly become a huge performance problem.
However, such an argument does not measure up to actual numbers.
 
A simple ``FORALL`` is often a whole order of magnitude faster than a cursor ``FOR`` loop.
In particular, for tables with 50,000-100,000 rows, the runtime of a ``FORALL`` statement is `typically 5-10%`_ of that of a cursor ``FOR`` loop.
We have consistently found at least an order of magnitude difference with a comparison script of the `PL/SQL Oracle User Group`_ for table inserts of up to a million rows.
For a million rows the speed-up was closer to 25 than 10 though.
 
With these results it seems to make sense to break up a cursor ``FOR`` loop when the number of separate IUD statements for each iteration is less than 10, which for most practical purposes implies that it is a good idea to use ``FORALL`` in almost all cases.
After all, with a 10x runtime improvement per IUD statement you need at least 10 individual statements per iteration to arrive at the same performance as with a single cursor ``FOR`` loop.
To avoid too bold a statement we rephrase it as that it is always a good idea to at least compare the performance of your ``FOR`` loops to an IUD ``FORALL``.
 
Important to note is that there is *no significant difference* in performance between a single SQL statement and a ``FORALL`` statement in our tests with up to a million rows.
 
Similarly, a ``BULK COLLECT`` obviates many context switches, thereby reducing the execution time.
Something that is important to keep in mind, though, is that filling a collection requires sufficient memory in the PGA.
As the number of simultaneous connections grows, so do the requirements on the overall PGA of the :term:`database instance <instance>`.
 
Ideally you'd figure out how much PGA memory you can consume and set the ``LIMIT`` clause accordingly.
However, in many situations you do not want to or cannot compute that number, which is why it is nice to know that a value of `at least 25`_ has been shown to improve the performance significantly.
Beyond that, the number is pretty much dependent on how much of a memory hog you want your collection to be.
Note that the ``LIMIT`` clause `does not control`_ (i.e. constrain) the PGA, it merely attempts to manage it more effectively.
In addition, the initialization parameter ``PGA_AGGREGATE_TARGET`` does not insulate your database from issues with allocating huge collections: the ``LIMIT`` option really is important.
 
The difference among various ``LIMIT`` values may be negligible when it comes to the runtime, in particular when there is enough PGA available, but it is noticeable when you look at the `memory consumed`_.
When there isn't enough memory available and you do not specify the ``LIMIT``, PGA swapping can cause the performance to degrade as well.
Beware!
 
For various tables with more than a few thousand records, a value between 100 and 1,000 for ``LIMIT``'s value seems to be `an apt initial choice`_.
Above 1,000 records the PGA consumption grows considerably and may cause scalability issues.
Below that value the benefit may be too small to notice, especially with smaller tables.
 
Table Functions
---------------
When multiple rows are inserted with individual ``INSERT ... VALUES`` statement, Oracle generates more redo than when using a single ``INSERT ... SELECT``.
This can lead to major differences in the overall runtime performance.
So, how can we benefit from set-based SQL rather than row-based PL/SQL when ``FORALL`` is not an option?
 
In Chapter 21 of `Oracle PL/SQL Programming`_ Feuerstein and Pribyl discuss the benefits of ``BULK COLLECT``-``FORALL`` (BCLFA) versus pipelined table functions (PTF).
`Their example`_ borrows heavily from Oracle's own `stock ticker example`_.
The essence of the problem is to retrieve data from a table, transform it, and then insert the records into another table.
The table function is merely used to take advantage of set-based SQL: ``INSERT INTO ... SELECT ... FROM TABLE( tab_func(...) )``.
 
The situation described can admittedly be handled with a classical ``BULK COLLECT ... LIMIT`` (BCL) and ``FORALL`` combination, but it could easily be extended to insert the data into multiple tables, which makes a simple single ``FORALL ... INSERT`` statement impossible.
 
So, what happens when we run the various versions?
Here's an overview of several runs on 12c, where we show the improvement factors in the execution time and redo generation compared to the baseline of the ``FOR`` loop to fetch the data and a pipelined table function to insert it:
 
+----------------------------------------------+-----------+--------+
| Method                                       | Execution | Redo   |
+==============================================+===========+========+
| ``FOR`` loop ``FETCH`` and ``INSERT``        | +3.5      | +7.3   |
+----------------------------------------------+-----------+--------+
| ``FOR`` loop ``FETCH`` → PTF ``INSERT``      | 0         | 0      |
+----------------------------------------------+-----------+--------+
| BCLFA                                        | -3.2      | (+7.3) |
+----------------------------------------------+-----------+--------+
| BCL ``FETCH`` → PTF ``INSERT``               | -2.7      | \(0\)  |
+----------------------------------------------+-----------+--------+
| BLC ``FETCH`` → parallel PTF ``INSERT``      | -5.3      | -1500  |
+----------------------------------------------+-----------+--------+
 
Notice the parentheses around the redo information for BCLFA and the BLC-PTF combination.
These numbers are typically close to the ones for the ``FOR`` loop (+7.3, i.e. a more than sevenfold increase in the amount of redo generated) and ``FOR``-PTF combination (0, i.e. no improvement at all), respectively.
The reason is that redo is obviously generated for IUD statements and in these cases the ``INSERT`` statements are identical to the ones mentioned.
Any differences are due to what comes before: a ``SELECT`` can generate redo too due to `block clean-outs`_.
The effect of block `clean-outs`_ is most obvious directly *after* IUD statements that affect many blocks in the database; the effect is usually relatively small.
So, depending on how you sequence your comparisons and what you do in-between, the numbers may be slightly different or even exactly the same.
 
These values obviously depend on the values used for the ``LIMIT`` clause in relation to the number of rows to be inserted, and the degree of parallelism, at least for the last entry.
It is clear that a parallel ``INSERT`` with a pipelined table function is the most efficient alternative.
The reason the redo generation is so low for that combination is that parallel inserts are :ref:`direct-path inserts <sql-hints-dpins>`.
For direct-path inserts, redo logging can be disabled.
 
Even without the parallel PTF ``INSERT``, the BCL is responsible for a threefold decrease of the execution time.
What is also obvious is that the cursor ``FOR`` loop is by far the worst option.
 
What should you take from all this?
 
Well, a cursor ``FOR`` loop is pretty much the worst choice, and it should only be a method of last resort.
Even though the same iteration can be used to extract, transform, and load the data one row at a time, it is slower than fetching it in bulk with ``BULK COLLECT ... LIMIT`` , then modifying it, and finally bulk-unloading it with ``FORALL`` .
 
When you are dealing with queries that return multiple records or rows, always use ``BULK COLLECT ... LIMIT`` (BCL).
In case you are faced with IUD statements and whenever a simple BCLFA is possible it is probably your best shot at getting a considerable performance improvement.
If, however, you require complex transformations and have multiple IUD statements, then a parallelized PTF may further drive down the cost of running your application logic.
Pipelined table functions are also a good choice when you are concerned about redo generation.

Caveats
=======
We have already mentioned that ``FORALL`` can only be used when one DML statement needs to be executed.
Unfortunately, that's not where the bad news ends.
 
Any *unhandled* exception causes *all* IUD changes to be rolled back.
The exception to this is when exceptions are managed with the ``SAVE EXCEPTIONS`` clause.
If there are failures that are saved, Oracle will raise a single exception (``ORA-24381``) for the entire statement after completing the ``FORALL``.
You can query the pseudo-collection ``SQL%BULK_EXCEPTIONS`` to get the information about these exceptions.
 
Please note that any collection index referenced in the IUD statement of a ``FORALL`` cannot be an expression: it must be the plain index itself.
The collection index is implicitly defined by Oracle as a ``PLS_INTEGER``.
 
Another gotcha is that `bulk SQL`_ can only be performed on local tables: you cannot do a ``BULK COLLECT`` over a database connection.
Furthermore, parallel DML is disabled for bulk SQL.
 
As of Oracle Database 10gR2, the PL/SQL compiler `automatically optimizes`_ most cursor ``FOR`` loops into constructs that run with performance comparable to a ``BULK COLLECT``.
Yay!
Unfortunately, this does *not* happen automatically when there are IUD statements in your ``FORALL`` statements: these statements require manual intervention.
Boo!

For *pipelined* table functions we can define the collection types they return inside a package specification.
Oracle automatically creates database (i.e. schema-level) types based on the record and collection types defined in the package specification.
The reason Oracle does this is that table functions are invoked by the SQL engine, which does not know about PL/SQL constructs inside the package.
As such, Oracle must ensure that the SQL engine can do its work, so it creates the `shadow types`_ implicitly.
These types typically have names that start with ``SYS_PLSQL_``.
So, while you have all your types in the same file (i.e. package specification), which makes maintaining your code base easier, you end up with system-generated identifiers for types, which in itself can be a maintenance issue.
  
.. _tune their PL/SQL code: http://docs.oracle.com/database/121/LNPLS/tuning.htm#LNPLS012
.. _in one go: http://www.orafaq.com/node/1399
.. _collections: http://www.oracle.com/technetwork/issue-archive/2012/12-jul/o42plsql-1653077.html
.. _Steven Feuerstein and Bill Pribyl: http://shop.oreilly.com/product/0636920024859.do
.. _counter-intuitive: http://docs.oracle.com/database/121/LNPLS/composites.htm#CJAJAGII
.. _collection shortcuts: http://docs.oracle.com/database/121/ARPLS/d_sql.htm#CHDEEDCH
.. _bounds clause: http://docs.oracle.com/database/121/LNPLS/forall_statement.htm#LNPLS01321
.. _rendered obsolete: http://www.oracle.com/technetwork/issue-archive/o54plsql-087195.html
.. _can be faster: http://www.dba-oracle.com/plsql/t_plsql_cursors.htm
.. _liar's paradox: http://en.wikipedia.org/wiki/Liar_paradox
.. _bulk SQL: http://www.oracle.com/technetwork/issue-archive/2012/12-sep/o52plsql-1709862.html
.. _typically 5-10%: http://www.oracle.com/technetwork/issue-archive/o14tech-plsql-091175.html
.. _at least 25: http://www.oracle.com/technetwork/issue-archive/2008/08-mar/o28plsql-095155.html
.. _PL/SQL Oracle User Group: http://psoug.org/reference/array_processing.html
.. _block clean-outs: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:44798632736844
.. _clean-outs: http://jonathanlewis.wordpress.com/2009/06/16/clean-it-up
.. _automatically optimizes: http://www.toadworld.com/platforms/oracle/b/weblog/archive/2011/07/13/quicktips-using-bulk-collect-with-queries.aspx
.. _does not control: http://plsql-challenge.blogspot.de/2011/07/use-limit-to-control-pga-in-session.html
.. _memory consumed: http://ricramblings.blogspot.de/2014/02/does-limit-clause-on-bulk-collectselect.html
.. _an apt initial choice: http://www.apress.com/9781430234852
.. _Oracle PL/SQL Programming: http://shop.oreilly.com/product/0636920024859.do
.. _Their example: http://www.oracle-developer.net/display.php?id=429
.. _stock ticker example: http://docs.oracle.com/database/121/ADDCI/pipe_paral_tbl.htm#ADDCI4691
.. _shadow types: http://oracle-base.com/articles/misc/pipelined-table-functions.php#implicit_types
