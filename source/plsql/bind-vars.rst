.. _plsql-bind-vars:

**************
Bind Variables
**************
Oracle Database developers drool when they hear the phrase "bind variables".
Well, not literally, but it is often cited as the key to solid application performance.
 
When SQL statements are littered with literals, the shared pool is likely to be cluttered with many similar statements.
When you execute the same statement over and over again but each time with different literals in the ``WHERE`` clause, you cause Oracle to parse the statement every time it is executed.
In many cases, particularly when the data isn't horribly skewed, the execution plan is the same.
Such a hard parse is a waste of valuable resources: the parsing itself costs CPU time and the various cursors bog up the shared pool.
 
"But what can we do?"
 
Use bind variables!
 
Straight from the `horse's mouth`_: "[a] bind variable is a placeholder in a SQL statement that must be replaced with a valid value or value address for the statement to execute successfully. By using bind variables, you can write a SQL statement that accepts inputs or parameters at run time."
 
That's right, a bind variable is a simple placeholder; it's the equivalent of "insert your literal here".
Whenever you change a literal, it does not matter: the statement is effectively the same.
Similarly, the execution plan remains as is.
Since there is no reason for Oracle to do a hard parse of a statement it still has in the inventory, Oracle merely looks up the plan.
 
PL/SQL Variables
================
Bind variables are related to host variables.
Host variables are defined in the host or caller, whereas bind variables accept values from the caller to SQL.
In PL/SQL the distinction between bind and host variables disappears.
 
Bind variables can be used in almost any place in PL/SQL with one exception: bind variables in anonymous PL/SQL blocks cannot appear in conditional compilation directives.
 
Variables in ``WHERE`` and ``VALUES`` clauses of *static* DML statements are automatically made bind variables in PL/SQL.
Before you run off and think that Oracle takes care of everything for you, hang on a minute.
That only applies to static statements; the emphasis in the previous sentence was intentional.
Dynamic SQL is slightly different.
 
Let's take a look at five common examples:
 
.. code-block:: sql
   :linenos:
 
    --
    -- Example 1: typically found in anonymous PL/SQL blocks
    --
    SELECT
      *
    BULK COLLECT INTO
      ...
    FROM
      fridge
    WHERE
      product = 'Beer';                -- hard-coded literal
 
    --
    -- Example 2: typically found in anonymous PL/SQL blocks
    --
    SELECT
      *
    BULK COLLECT INTO
      ...
    FROM
      fridge
    WHERE
      product = :prod;                 -- bind variable
 
    --
    -- Example 3: typically found in PL/SQL modules
    --
    SELECT
      *
    BULK COLLECT INTO
      ...
    FROM
      fridge
    WHERE
      product = product_in;            -- bind variable
 
    --
    -- Example 4: typically found in PL/SQL modules
    --
    EXECUTE IMMEDIATE
      'SELECT * FROM fridge ' ||
      'WHERE product = ''' ||
      product_in || ''''               -- literal: value is embedded in statement
      BULK COLLECT INTO ...
    ;          
 
    --
    -- Example 5: typically found in PL/SQL modules
    --
    EXECUTE IMMEDIATE
      'SELECT * FROM fridge ' ||
      'WHERE product = :prod'          -- bind variable, because...
      BULK COLLECT INTO ...
      USING product_in                 -- ... we substitute rather than embed the value
    ;
 
Hopefully, no one would build dynamic SQL like that as it is open to SQL injections; the package ``DBMS_ASSERT`` offers some basic sanity checks on raw (user) input.
The code is only shown for the purposes of our demonstration of the various options with regard to literals and bind variables.
You can  use ``DBMS_SQL`` as an alternative to dynamically build a SQL statement, but we have decided not to show the code for reasons of brevity.
 
There is sometimes `a good reason`_ to utilize ``DBMS_SQL`` instead of native dynamic SQL (NDS).
NDS has to be parsed every time the statement is executed; for complex statements the overhead can be significant.
Parsing can be bypassed with ``DBMS_SQL``.
For instance, when a statement is executed for different values inside a loop, you just have to place the call to ``DBMS_SQL.PARSE`` outside of the loop; the calls to ``DBMS_SQL.BIND_VARIABLE`` need to be placed inside the loop.
 
Bind Peeking
============
If bind variables are so grand, why not enable them by default, everywhere?
 
The problem lies in what is referred to as bind peeking.
When Oracle encounters a statement with bind variables for the very first time, it looks at the literals supplied, checks the histogram (if available), and then fixes the execution plan.
 
In itself this seems reasonable, right?
When data is highly skewed that may become an issue.
Let's go back to our ``fridge`` table and fill it in accordance with the appetite of many developers: beer, beer, and some more beer.
Because we have a few guests over tonight we also buy some white wine, salad, and avocados; please, don't ask why.
We have created a histogram too: 95% of the rows are related to beer, whereas the remaining 5% are split among the three newly acquired products.
 
When we send our friendly household robot to run to the kitchen to get beer, and the query contains a bind variable for product, a full table scan will be used.
The next time we send it to get white wine and it still uses a full table scan even though an index lookup would be much faster.
Why does it do a full table scan?
For the first execution our database robot peeked at the bind variable's value, and because its value was ``'Beer'``, the execution plan was fixed with a full table scan.
The second execution did not care about the fact that we wanted ``'White Wine'`` since the robot had already decided that the execution plan involved a full table scan.
 
The reverse situation is also possible.
An index scan is used based on the initial execution of the request for something other than beer, even though a full table scan is much more advantageous when we desire beer.
Thus, as stated by `Arup Nanda`_, "smart SQL coders will choose when to break the cardinal rule of using bind variables, employing literals instead."
 
Adaptive Cursor Sharing and Adaptive Execution Plans
====================================================
That's not the end of the story though.
As of 11g, Oracle has introduced the concept of adaptive cursor sharing.
Based on the performance of a SQL statement, the execution plan may be marked for revision the next time the statement is executed, even when the underlying statistics have not changed at all.
 
In ``v$sql`` this is indicated by the columns ``is_bind_sensitive`` and ``is_bind_aware``.
The former indicates that a particular ``sql_id`` is a candidate for adaptive cursor sharing, whereas the latter means that Oracle acts on the information it has gathered about the cursor and alters the execution plan.
 
Problematic is that adaptive cursor sharing can only lead to an improved plan *after* the SQL statement has `tanked at least once`_.
You can bypass the initial monitoring by supplying the ``BIND_AWARE`` hint: it instructs the database that the query is bind sensitive and adaptive cursor sharing should be used from the very first execution onwards.
A prerequisite for the hint to be used is that the bind variables only appear in the ``WHERE`` clause and an applicable histogram is available.
The hint may improve the performance but you should be aware that it's rarely the answer in the case of :ref:`generic static statements <plsql-smart-logic>`, which we describe below.
The ``NO_BIND_AWARE`` hint does exactly the opposite: it disables bind-aware cursor sharing.
 
Frequency histograms are important to adaptive cursor sharing.
The problem is that they are `expensive to compute, unstable when sampled, and the statistics have to be collected at the right time`_.
In Oracle Database 12c, the speed with which histograms are collected has been `greatly improved`_.
 
Adaptive cursor sharing has a slight overhead though, as explained by the `Oracle Optimizer team`_: additional cursor memory, more soft and hard parses, and more child cursors.
The last one may cause cursor `cache contention`_.
 
The default setting for the ``CURSOR_SHARING``  parameter is ``'EXACT'``.
You can also set it to ``'FORCE'`` (`11g and 12c`_) or ``'SIMILAR'`` (`11g`_).
These settings are, however, generally recommended only as a temporary measure.
`Oracle's own recommendation`_ boils down to the following: ``'FORCE'`` is only used by lazy developers who cannot be bothered to use bind variables.
Please note that Oracle *never* replaces literals in the ``ORDER BY`` clause because the ordinal notation affects the execution plan: cursors with different column numbers in the ``ORDER BY`` cannot be shared.
 
Adaptive execution plans were introduced in 12c.
When we talked about execution plans we already mentioned the :ref:`mechanics <sql-adaptive>`: they allow execution plans to be modified on the fly.
On a development or test system adaptive cursor sharing and adaptive execution plans may mask underlying problems that need to be investigated and solved before the code hits production, which is why they should be switched off.
There are even some people who believe that these features have no place in a production system either because once you have determined the optimal execution plan, it should not be changed, lest you run into unexpected surprises.
As such, untested execution plans should never be released into the wild, according to `Connor McDonald and Tony Hasler`_.
 
.. _plsql-smart-logic:
 
Generic Static Statements
=========================
Developers sometimes write static SQL statements that are very generic in the sense that they accept different sets of parameters that are bound at runtime; some may be supplied, others may not.
The reason people go for such static code rather than a dynamic solution may be based on the misguided idea that dynamic SQL is slow.
This is what is referred to by `Markus Winand`_ as smart logic, probably because developers think they have created a nice, generic template.
 
An example might be as follows:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 11-13
 
    SELECT
      manufacturer
    , product
    , temperature
    , MIN(expiry_date) AS min_expiry_date
    , MAX(expiry_date) AS max_expiry_date
    , COUNT(*)             AS num_items
    FROM
      fridge
    WHERE
        ( product      = :prod OR :prod IS NULL )
    AND ( temperature  = :temp OR :temp IS NULL )
    AND ( manufacturer = :manu OR :manu IS NULL )
    GROUP BY
      manufacturer
    , product
    , temperature
    ;
 
In itself the query looks nice: it elegantly deals with many different cases thanks to the use of bind variables.
Swell!
 
The genericness of such a SQL statement is a fabulous from a coder's point of view.
From a performance perspective it is a tremendous weakness.
The reason is that the execution plan is the same in all instances.
 
Suppose for example that there is an index on ``manufacturer`` and ``product``.
 
**Question 1**: Will the index be used when the manufacturer of items in the fridge is given?
 
**Answer 1**: Unfortunately, no.
 
An index scan may be much more beneficial but Oracle does not know that.
There are 8 possible combinations of bind parameter values and there is no way that Oracle can capture the best plan in each of these cases with a single execution plan, especially since some predicates can benefit from the index whereas other cannot.
Hence, Oracle decides to do a full table scan and filter rows.
 
**Question 2**: Will the index be used when both manufacturer (``:manu``)  and product (``:prod``) are provided?
 
**Answer 2**: Nope.
 
An ``OR`` confounds index use.
Apart from that, we have already established that Oracle does not have a customized execution plan for this case, so that it reverts to a full table scan anyway.
 
**Question 3**: Will an index skip scan be used when only the product is supplied at runtime?
 
**Answer 3**: Again, no.
 
**Conclusion**: Oracle will *never* use an index for our query.
The issue with such generic pieces of static code is that the one-size-fits-all code leads to a one-plan-suits-(almost)-none situation.
The only cases where full table scans are *always* appropriate are when either only the temperature is known at runtime or all items from the fridge need to be returned.
Since ``temperature`` is not in our index it cannot benefit from an the index anyway.
For some special values of ``manufacturer`` and ``product`` a full table scan may also be the best solution, but it is doubtful that this applies to all possible cases.
 
The solution is to use dynamic SQL with bind variables and separate each case.
Alternatively, you can write a function that executes different SQL statements with bind variables based on the input of the function.
This entails that you will have a few more execution plans in the shared pool, but they are at least tuned to each instance rather than bad for all.
 
.. _horse's mouth: http://docs.oracle.com/database/121/TGSQL/tgsql_cursor.htm#TGSQL849
.. _a good reason: http://www.oracle.com/technetwork/issue-archive/o64sql-095035.html
.. _Arup Nanda: http://www.oracle.com/technetwork/articles/sql/11g-sqlplanmanagement-101938.html
.. _expensive to compute, unstable when sampled, and the statistics have to be collected at the right time: http://allthingsoracle.com/histograms-part-1-why
.. _greatly improved: http://jonathanlewis.wordpress.com/2013/07/14/12c-histograms
.. _tanked at least once: http://www.toadworld.com/platforms/oracle/b/weblog/archive/2014/03/04/oracle-12c-bind-variable-tips.aspx
.. _Oracle Optimizer team: http://blogs.oracle.com/optimizer/entry/how_do_i_force_a
.. _cache contention: http://dba-oracle.com/t_oracle_library_cache_contention_tips.htm
.. _Oracle's own recommendation: http://docs.oracle.com/database/121/TGSQL/tgsql_cursor.htm#TGSQL94750
.. _Connor McDonald and Tony Hasler: http://www.apress.com/9781430259770
.. _11g and 12c: http://docs.oracle.com/database/121/REFRN/refrn10025.htm#REFRN10025
.. _11g: http://docs.oracle.com/cd/B28359_01/server.111/b28320/initparams041.htm#REFRN10025
.. _Markus Winand: http://use-the-index-luke.com/sql/where-clause/obfuscation/smart-logic
