.. _plsql-loops-performance:
 
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

What happens in the lifetime of a cursor?

#. Open: Private memory is allocated.
#. Parse: The SQL statement is parsed.
#. Bind: Placeholders are replaced by actual values (i.e. literals).
#. Execute: The SQL engine executes the statement and sets the record pointer to the very first record of the result set.
#. Fetch: A record from the current position of the record pointer is obtained after which the record pointer is incremented.
#. Close: Private memory is de-allocated and released back to the SGA.

All steps are done in the background for implicit cursors. 
For explicit cursors, the first four stages are included in the ``OPEN`` statement. 
 
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
| BCL ``FETCH`` → parallel PTF ``INSERT``      | -5.3      | -1500  |
+----------------------------------------------+-----------+--------+
 
Notice the parentheses around the redo information for BCLFA and the BCL-PTF combination.
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

In the valuable yet footnote-heavy `Doing SQL from PL/SQL`_, Bryn Llewellyn notes a factor of 3 difference in the runtime performance between all ``DBMS_SQL`` calls inside a ``FOR`` loop and the same construct but with ``OPEN_CURSOR``, ``PARSE``, ``DEFINE_COLUMN``, and ``CLOSE_CURSOR`` outside of the loop on a test data set of 11,000 rows.
The difference is obviously in favour of the latter alternative. 
A simple cursor ``FOR`` loop is about twice as fast as the best ``DBMS_SQL`` option, with no noticeable difference when the single DML statement inside the loop is replaced with an ``EXECUTE IMMEDIATE`` (NDS) solution.
 
An upsert (i.e. ``INSERT`` if record does not exist, otherwise ``UPDATE``) is best done with a ``MERGE`` rather than a `PL/SQL-based solution`_ with ``UPDATE ... SET ROW ...`` that uses a ``DUP_VAL_ON_INDEX`` exception to handle the update in case the entry for the primary key already exists, as the ``MERGE`` runs noticeably faster.

What should you take from all this?
 
Well, a cursor ``FOR`` loop is pretty much the worst choice, and it should only be a method of last resort.
Even though the same iteration can be used to extract, transform, and load the data one row at a time, it is slower than fetching it in bulk with ``BULK COLLECT ... LIMIT`` , then modifying it, and finally bulk-unloading it with ``FORALL`` .
 
When you are dealing with queries that return multiple records or rows, always use ``BULK COLLECT ... LIMIT`` (BCL).
In case you are faced with IUD statements and whenever a simple BCLFA is possible it is probably your best shot at getting a considerable performance improvement.
If, however, you require complex transformations and have multiple IUD statements, then a parallelized PTF may further drive down the cost of running your application logic.
Pipelined table functions are also a good choice when you are concerned about redo generation.

.. _`rendered obsolete`: http://www.oracle.com/technetwork/issue-archive/o54plsql-087195.html
.. _`can be faster`: http://www.dba-oracle.com/plsql/t_plsql_cursors.htm
.. _`liar's paradox`: http://en.wikipedia.org/wiki/Liar_paradox
.. _`typically 5-10%`: http://www.oracle.com/technetwork/issue-archive/o14tech-plsql-091175.html
.. _`at least 25`: http://www.oracle.com/technetwork/issue-archive/2008/08-mar/o28plsql-095155.html
.. _`PL/SQL Oracle User Group`: http://psoug.org/reference/array_processing.html
.. _`block clean-outs`: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:44798632736844
.. _`clean-outs`: http://jonathanlewis.wordpress.com/2009/06/16/clean-it-up
.. _`does not control`: http://plsql-challenge.blogspot.de/2011/07/use-limit-to-control-pga-in-session.html
.. _`memory consumed`: http://ricramblings.blogspot.de/2014/02/does-limit-clause-on-bulk-collectselect.html
.. _`an apt initial choice`: http://www.apress.com/9781430234852
.. _`Oracle PL/SQL Programming`: http://shop.oreilly.com/product/0636920024859.do
.. _`Their example`: http://www.oracle-developer.net/display.php?id=429
.. _`stock ticker example`: http://docs.oracle.com/database/121/ADDCI/pipe_paral_tbl.htm#ADDCI4691
.. _`Doing SQL from PL/SQL`: http://www.oracle.com/technetwork/database/features/plsql/overview/doing-sql-from-plsql-129775.pdf
.. _`PL/SQL-based solution`: http://psoug.org/definition/ROW.htm
