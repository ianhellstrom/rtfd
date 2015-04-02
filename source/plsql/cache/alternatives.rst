.. _plsql-cache-alternatives:
 
Alternatives
============
Oracle offers a few alternatives for caching: deterministic functions, package-level variables, and the result cache (for functions).
Each of these techniques has its pros and cons.
 
Package-level variables are conceptually the simplest but require the developer to think about and manage the cache.
A package-level variable is really what it says on the box: a variable in a package that contains static (reference) data that needs to be accessed many times.
 
Suppose you have an expensive deterministic function that you need to execute many times.
You can run the function for different parameter values and store the combination of input-output as a key-value pair in a collection at the package level.
You can thus access the data without having to recompute it every time.
 
Similarly, you can store (pre-aggregated) data from a table that does not change while you work with it, so you don't have to go back and forth between the disk and the PGA.
Alternatively, you can create a just-in-time cache, where each time a new value or row is requested, you check the cache and return the data (if already present) or add it to the cache (if not yet available in the cache) for future use.
That way, you do not create a huge collection up front but improve the lookup performance of subsequent data access requests or computations.
 
Another common use case for package-level variables that has little to do with caching though is when people try to avoid the dreaded `mutating table error with triggers`_.
 
A problem with package-level variables is that they are stored in the :term:`UGA`.
This means that the memory requirements go up as there are more sessions.
Of course, you can add the ``SERIALLY_REUSABLE`` directive to the package to reduce storage requirements, but it also means that the data needs to be exactly the same across sessions, which is rare, especially in-between transactions.
 
You do *not* want to use package-level variables as caches when the data contained in them changes frequently during a session or requires too much memory.
A measly 10 MB per user quickly becomes 1 GB when there are 100 sessions, making a what seemed to be a smart idea a veritable memory hog.
 
In terms of speed, packaged-based caching beats the competition, although the function result cache is `a close second`_.
But, as we have mentioned, package-based caching is also fairly limited in its uses and requires a developer to think carefully about the caching mechanism itself and the overall memory requirements.
Moreover, on Oracle Database 11g and above, the built-in alternatives are almost always a better option.
 
``DETERMINISTIC`` Functions
---------------------------
The ``DETERMINISTIC`` clause for functions is ideal for functions that do not have any non-deterministic components.
This means that each time you provide the function with the same parameter *values*, the result is the same.
When you define a function you can simply add the ``DETERMINISTIC`` option to the declaration section, making sure that the function (or any functions or procedures it calls) does not depend on the state of session variables or schema objects as the `results may vary across invocations`_.
This option instructs the optimizer that it may use a cached result whenever it encounters a previously calculated result.
 
Any speed-up from specifying a function as ``DETERMINISTIC`` becomes apparent when you execute the same function many times with the same parameter values *in the same SQL statement*.
A typical example is a user-defined conversion function that is called for each row of the result set of a query
 
Function-based indexes can only use functions marked ``DETERMINISTIC``.
The same goes for materialized views with ``REFRESH FAST`` or ``ENABLE QUERY REWRITE``.
 
One restriction is that you cannot define a nested function as deterministic.
 
Whenever a function is free of side effects and deterministic, it is a good practice to add the ``DETERMINISTIC`` option.
Whether the optimizer makes use of that information is entirely up to Oracle, but fellow coders will know that you intended to create a deterministic, side-effect free function, even when Oracle does not see a use for that information.
 
'What happens when I label a function as ``DETERMINISTIC`` but it secretly isn't?'
 
First off, why on earth would you do that?!
Second, Oracle cannot fix stupidity.
Its capability to discover non-deterministic ``DETERMINISTIC`` functions is not just limited, it is non-existent.
Yes, that's right: Oracle does not check whether you are telling the truth.
It trusts you implicitly.
 
What can happen, though, is that a user sees dirty (i.e. uncommitted) data because the incorrectly-marked-as-deterministic function queries data from a table that has been cached inappropriately.
`Neil Chandler`_ has written about the multi-version concurrency control model (MVCC) used by Oracle Database, which explains the roles of isolation levels, REDO, and UNDO with regard to `dirty reads`_, in case you are interested.
If you tell Oracle that a function is deterministic even though it is not, then the results can be unpredictable.
 
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
 
``DETERMINISTIC`` vs ``RESULT_CACHE``
-------------------------------------
A common question with caching is whether the ``DETERMINISTIC`` option or the ``RESULT_CACHE`` is best.
As always, the answer is: 'It depends.'
 
When you call a deterministic function many times from within the *same* SQL statement, the ``RESULT_CACHE`` does not add much to what the ``DETERMINISTIC`` option already covers.
Since a single SQL statement is executed from only one session, the function result cache cannot help with multi-session caching as there is nothing to share across sessions.
As we have said, marking a deterministic function as ``DETERMINISTIC`` is a good idea in any case.
 
When you call a deterministic function many times from *different* SQL statements — in potentially different sessions or even `instances of a RAC`_ — and even PL/SQL blocks, the ``RESULT_CACHE`` does have benefits.
Now, Oracle can access a single source of cached data across statements, subprograms, sessions, or even application cluster instances.
 
The 'single source of cached data' is of course only true for DR units.
For IR units, the function result cache is user-specific, which probably dampens your euphoria regarding the function result cache somewhat.
Nevertheless, both caching mechanisms are completely handled by Oracle Database.
All you have to do is add a simple ``DETERMINISTIC`` and/or ``RESULT_CACHE`` to a function's definition.
 
.. _`a close second`: http://www.oracle-developer.net/display.php?id=504
.. _`mutating table error with triggers`: http://oracle-base.com/articles/9i/mutating-table-exceptions.php#solution_1
.. _`results may vary across invocations`: http://docs.oracle.com/database/121/LNPLS/function.htm
.. _`Neil Chandler`: http://chandlerdba.wordpress.com/2013/12/01/oracles-locking-model-multi-version-concurrency-control
.. _`dirty reads`: http://docs.oracle.com/database/121/CNCPT/consist.htm
.. _`per user`: http://www.oracle.com/technetwork/issue-archive/2013/13-sep/o53plsql-1999801.html
.. _`instances of a RAC`: http://www.oracle.com/technetwork/articles/datawarehouse/vallath-resultcache-rac-284280.html
.. _`official documentation`: http://docs.oracle.com/database/121/REFRN/stats002.htm
.. _`Adrian Billington`: http://www.oracle-developer.net/display.php?id=504
.. _`several`: http://www.oracle.com/technetwork/issue-archive/2010/10-sep/o57plsql-088600.html
.. _`solutions`: http://www.oracle-developer.net/utilities.php
.. _`a helping hand`: http://www.dba-oracle.com/oracle11g/oracle_11g_result_cache_sql_hint.htm
.. _`high degree of concurrency`: http://www.pythian.com/blog/oracle-11g-result-cache-in-the-real-world
.. _`from 11gR1 to 11gR2`: http://afatkulin.blogspot.de/2010/06/11gr2-result-cache-scalability.html
.. _`latch contention`: http://www.toadworld.com/platforms/oracle/w/wiki/382.optimizing-the-oracle-11g-result-cache.aspx