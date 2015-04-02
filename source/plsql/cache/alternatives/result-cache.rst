.. _plsql-cache-alt-rc:

The ``RESULT_CACHE`` Option
---------------------------
As of Oracle Database 11g, the function result cache has entered the caching fray.
It offers the benefits of just-in-time package-level caching (and more!) but without the hassle.
All you have to do is add the ``RESULT_CACHE`` option to the function declaration section and that's it.
It couldn't be much easier!
 
The function result cache is ideal for data from tables that are queried from more frequently than they are modified, for instance lookup tables and materialized views (between refreshes).
When a table's data changes every few seconds or so, the function result cache may hurt performance as Oracle needs to fill and clear the cache before the data can be used many times.
On the other hand, when a table (or materialized view) changes, say, every 10 minutes or more, but it is queried from hundreds or even thousands of times in the meantime, it can be beneficial to cache the data with the ``RESULT_CACHE`` option.
Recursive functions, small lookup functions, and user-defined functions that are called repeatedly but do not fetch data from the database are also ideal candidates for the function result cache.
 
With Oracle Database 11gR2, the ``RELIES ON`` clause is deprecated, which means that you don't even have to list the dependencies: Oracle will figure everything out for you!
 
The database does *not* cache SQL statements contained in your function.
It 'merely' caches input values and the corresponding data from the ``RETURN`` clause.
 
Oracle manages the function result cache in the :term:`SGA`.
In the background.
Whenever changes are committed to tables that the cache relies on, Oracle automatically invalidates the cache.
Subsequent calls to the function cause the cache to be repopulated.
Analogously, Oracle ages out cached results whenever the system needs more memory, so you, the database developer, are completely relieved of the burden of designing, developing, and managing the cache.
 
Since the function result cache is in the SGA rather than the :term:`PGA`, it is somewhat slower than PGA-based caching.
However, if you have hidden ``SELECT`` statements in functions, the SGA lookups thanks to the function result cache beat any non-cached solutions with context switches hands down.
 
Sounds too good to be true?
 
It is.
 
First, the function result cache only applies to stored functions not functions defined in the declaration section of anonymous blocks.
Second, the function cannot be a pipelined table function.
Third, the function cannot query from data dictionary views, temporary tables, ``SYS``-owned tables, sequences, or call any non-deterministic PL/SQL function.
Moreover, pseudo-columns (e.g. ``LEVEL`` and ``ROWNUM``) are prohibited as are ``SYSDATE`` and similar time, context, language (NLS), or GUID functions.
The function has to be free of side effects, that is, it can only have ``IN`` parameters; ``IN OUT`` and ``OUT`` parameters are not allowed.
Finally, ``IN`` parameters cannot be a ``LOB``, ``REF CURSOR``, collection, object type, or record.
The ``RETURN`` type can likewise be none of the following: ``LOB``, ``REF CURSOR``, an object type, or a record or collection that contains a ``LOB``, ``REF CURSOR``, and/or an object type.
 
The time to look up data from the function result cache is on par with a context switch or a function call.
So, if a PL/SQL function is almost trivial *and* called from SQL, for instance a simple concatenation of ``first_name`` and ``last_name``, then the function result cache solution may be slower than the same *uncached* function.
 
Inlining, or rather hard coding, of simple business rules seems to be even faster as demonstrated by `Adrian Billington`_, although we hopefully all agree that hard coding is a bad practice, so we shall not dwell on these results and pretend they never existed.
 
Beware that the execution plan of a SQL statement does not inform you that a function result cache can or even will be used in clear contrast to the query result cache.
The reason is both simple and obvious: ``RESULT_CACHE`` is a PL/SQL directive and thus not known to the SQL engine.

Latches
^^^^^^^
The result cache is protected by a single :term:`latch`, the so-called result cache (RC) latch.
Since latches are serialization devices, they typically stand in the way of scalability, especially in environments with a `high degree of concurrency`_, such as OLTP applications.

Because there is only one latch on the result cache, only one session can effectively create fresh result cache entries at any given moment.
A high rate of simultaneous changes to the result cache are therefore detrimental to the performance of a database.
Similarly, setting the parameter ``RESULT_CACHE_MODE`` to ``FORCE`` is a guarantee to bring a database to its knees, as every single SQL statement will be blocking the RC latch.

Scalability issues have dramatically improved `from 11gR1 to 11gR2`_, but `latch contention`_ still remains an issue when rapidly creating result sets in the cache.

It should be clear that the function result cache only makes sense for relatively small result sets, expensive SQL statements that do not experience high rates of concurrent execution, and SQL code that is against relatively static tables.

IR vs DR Units
^^^^^^^^^^^^^^
The default mode of PL/SQL units is to run with the definer's rights (DR).
Such units can benefit from the function result cache without further ado.
Invoker's rights (IR) subprograms, created with the ``AUTHID CURRENT_USER`` rather than ``AUTHID DEFINER``, cannot use the function result cache, and an attempt at compilation leads to a ``PLS-00999`` error, at least prior to DB12c.
The reason is that a user would have been able to retrieve data cached by another user, to which the person who originally requested the data should not have access because its privileges are not sufficient.
 
This restriction has been lifted with 12c, and the security implications have been resolved.
The solution to the security conundrum is that the function result cache is `per user`_ for IR units.
This means of course that the ``RESULT_CACHE`` option  is only useful for functions that the same user calls many times with the same input values.
 
Memory Consumption
^^^^^^^^^^^^^^^^^^
That's all very nice, but how much memory does the function result cache gobble up?
 
A DBA can run ``EXEC DBMS_RESULT_CACHE.MEMORY_REPORT(detailed => true)`` to see detailed information about the memory consumption.
However, the purpose of these pages is to help fellow developers to learn about optimization techniques, which means
that ``DBMS_RESULT_CACHE`` is out of the question.
 
You can check the UGA and PGA memory consumption by looking at the data for your session from the following query:
 
.. code-block:: sql
   :linenos:
  
   SELECT 
     * 
   FROM 
     v$sesstat
   NATURAL JOIN
     v$statname
   ;
 
You can provide the name of the statistic you're interested in.
A full list of statistics can be found in the `official documentation`_.
For example, ``'session uga memory'`` or ``'session pga memory'``.
These are current values, so you'd check the metrics *before* and *after* you run your function a couple of times to see the PGA and UGA memory consumption of your function.
Obviously, there will be no (or very little) PGA consumption in the case of the function result cache.
 
There are also `several`_ `solutions`_ available that calculate the various statistics for you.
They typically work by checking the metrics before running a function several times, then run the function, after which they check the metrics again.
 
In case you need help configuring the function result cache, here's `a helping hand`_.

.. _`Adrian Billington`: http://www.oracle-developer.net/display.php?id=504
.. _`high degree of concurrency`: http://www.pythian.com/blog/oracle-11g-result-cache-in-the-real-world
.. _`from 11gR1 to 11gR2`: http://afatkulin.blogspot.de/2010/06/11gr2-result-cache-scalability.html
.. _`latch contention`: http://www.toadworld.com/platforms/oracle/w/wiki/382.optimizing-the-oracle-11g-result-cache.aspx
.. _`per user`: http://www.oracle.com/technetwork/issue-archive/2013/13-sep/o53plsql-1999801.html
.. _`official documentation`: http://docs.oracle.com/database/121/REFRN/stats002.htm
.. _`several`: http://www.oracle.com/technetwork/issue-archive/2010/10-sep/o57plsql-088600.html
.. _`solutions`: http://www.oracle-developer.net/utilities.php
.. _`a helping hand`: http://www.dba-oracle.com/oracle11g/oracle_11g_result_cache_sql_hint.htm