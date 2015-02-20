.. _sql-exec-plan:

***************
Execution Plans
***************
What happens to your SQL statement when you hit execute?

First, Oracle checks you statement for any glaring errors.
The syntax check verifies whether the language elements are sequenced correctly to form valid statements.
If you have neither made any typos in keywords and the like nor sequenced the language elements improperly, you're good for the next round.
Now Oracle moves on to evaluate the meaning of your syntactically legal code, which is known as semantic analysis. 
All references to database objects and host variables are scrutinized by Oracle to make sure they are valid. 
Oracle also inspects whether you are authorized to access the data.
These checks expand views referenced by your statement into separate query blocks.
For more details we refer you to the chapter *Syntactic and Semantic Checking* of the *Programmer's Guide to the Oracle Precompilers* for your `database version`_.

Once your SQL statement has passed both checks with flying colours, your statement receives a SQL ID and (MD5) hash value. 
The hash value is based on the first `few hundred characters`_ of your statement, so hash collisions can occur, especially for long statements.

You can find out the SQL ID and hash value of your SQL statement by querying ``v$sql``. 
To make life easier it is often best to add a comment unique to your statement, for instance ``SELECT /* my_custom_comment */ select_list FROM tab_name``.
Then you can simply look for your query from ``v$sql``:

.. code-block:: sql
   :linenos:
   
   SELECT 
       sql_id
     , hash_value
     , plan_hash_value
     , sql_text
   FROM
     v$sql
   WHERE
     sql_text LIKE 'SELECT /* my_custom_comment */%'
   ;

In case you happen to know the SQL ID already and would like to know the corresponding hash value, you can use the function ``DBMS_UTILITY.SQLID_TO_SQLHASH``, which takes the sql_id (``VARCHAR2``) and returns a ``NUMBER``. 

.. note::
   *All* characters affect the hash value, including spaces and line breaks as well as capitalization and of course comments.

The last stage of the parser is to look for possible shortcuts by sifting through the :term:`shared pool`, which is a "portion of the SGA that contains shared memory constructs such as shared SQL areas", which hold the parse tree and execution plans for SQL statements; each unique statement has only one shared SQL area.
We can distinguish two cases: `hard and soft parses`_.

#. **Soft parse** (library cache hit): if the statement hashes to a value that is identical to one already present in the shared pool *and* the texts of the matching hash values are the same *and* its parsed representation can be shared, Oracle looks up the execution plan and executes the statement accordingly.
   Literals must also be the same for Oracle to be able to use the same shared SQL area. 
   The exception is when ``CURSOR_SHARING`` is set to ``FORCE``.

#. **Hard parse** (library cache miss): if the statement has a hash value different from the ones that are available in the SGA *or* its parsed representation cannot be shared, Oracle hands the code over to the query optimizer.
   The query optimizer then has to build an executable version from scratch.

Criteria for when a SQL statement or PL/SQL block can be shared are described in the *Oracle Database Performance Tuning Guide*, which can be found `here <http://docs.oracle.com/cd/E16655_01/server.121/e15857/tune_shared_pool.htm#TGDBA564>`_. 
Basically, the statements' hashes and texts, all referenced objects, any bind variables (name, data type, and length), and the session environments have to match. PL/SQL blocks that do not use bind variables are said to be not re-entrant, and they are always hard-parsed.
To find out why statements cannot be shared you can use the view ``v$sql_shared_cursor``.
 
Perhaps you noticed that we had sneaked in the column ``plan_hash_value`` in the ``v$sql`` query above. 
SQL statements with different hash values can obviously have the `same plan`_, which means that their plan hash values are equal.

.. note::
   The plan hash value is merely an `indicator`_ of similar operations on database objects: filter and access predicates, which we shall discuss in more detail, are not part of the plan hash value calculation.

For hard parses, the next station on the :term:`SQL compiler` line is the query optimizer.
The query optimizer, or just optimizer, is the "built-in database software that determines the most efficient way to execute a SQL statement". 
The optimizer is also known as the cost-based optimizer (CBO), and it consists of the query transformer, the estimator, and the plan generator:

* The query transformer "decides whether to rewrite a user query to generate a better query plan, merges views, and performs subquery unnesting".
* The estimator "uses statistics [from the data dictionary] to estimate the selectivity, cardinality, and cost of execution plans. 
  The main goal of the estimator is to estimate the overall cost of an execution plan".
* The plan generator "tries out different possible plans for a given query so that the query optimizer can choose the plan with the lowest cost. It explores different plans for a query block by trying out different :term:`access paths <access path>`, join methods, and join orders". The optimizer also evaluates expressions, and it can convert correlated subqueries into equivalent join statements or vice versa.

What the optimizer does, in a nutshell, is apply fancy heuristics to figure out the best way to execute your query: it calculates alternate routes from your screen through the database back to your screen, and picks the best one.
By default Oracle tries to minimize the `estimated resource usage`_ (i.e. maximize the throughput), which depends on I/O, CPU, memory, the number of rows returned, and the size of the initial data sets.
The objective of the optimization can be altered by changing the value of ``OPTIMIZER_MODE`` parameter.

If you can recall our :ref:`example <sql-basics-proc-order>` of the robot, the beer, and the packet of crisps, you may remember that the robot had to check both the pantry and the fridge.
If we equip our robot with Oracle's query optimizer, the robot will not simply walk to the kitchen, find the items by searching for them, and then return with our refreshments, but try to do it as efficiently as possible.
It will modify our query without altering the query's function (i.e. fetch you some booze and a few nibbly bits), and explore its options when it comes to retrieving the items from the kitchen.
For instance, if we happen to have a smart fridge with a display on the door that shows where all bottles are located, including the contents, temperature, and size of each bottle, the robot does not have to rummage through decaying fruit and vegetables at the bottom in the hope that a bottle is located somewhere underneath the rubbish.
Instead it can look up the drinks (from an index) and fetch the bottles we want by removing them from the spots highlighted on the display (by ROWID).
What the optimizer can also figure out is whether it is more advantageous to grab the beers and then check the pantry, or the other way round (join order).
If the robot has to go down a few steps to obtain crisps while still holding the beer in its hands, the optimizer may decide that carrying the heavy and/or many beverages may be inefficient.
Furthermore, it may decide to pick up a tray and place the products it has already extracted on it (temporary table) while continuing its search for the remaining items.
Because the optimizer evaluates expressions it will know whether or not it has to do something: a predicate like ``0 = 1`` will be immediately understood by the optimizer, so that our friendly robot knows it has to do bugger all beforehand.

After the optimizer has given its blessings to the optimal execution plan, the row source generator is let loose on that plan. The row source generator produces an iterative plan, which is known as the `query plan`_. The query plan is a binary program that produces the result set when executed. 
It is structured as a row source tree, where a row source is the combination of a set of rows returned by a step in the execution plan and "a control structure that can iteratively process the rows", that is one row at a time. The row source tree shows an ordering of the tables, an access method for each table, a join method for tables affected by join operations, and data operations (filter, sort, aggregation, etc.)

During execution, the SQL engine executes each row source in the tree produced by the row source generator. This step is the only mandatory step in DML processing.

More information on the optimizer can be found on the `Oracle Technology Network`_ in the Database Concepts documentation under the the section *SQL* and the subsection *Overview of the Optimizer*.

.. _`database version`: http://www.oracle.com/technetwork/documentation/index.html#database
.. _`few hundred characters`: http://www.dba-oracle.com/concepts/hashing.htm
.. _`indicator`: http://oracle-randolf.blogspot.de/2009/07/planhashvalue-how-equal-and-stable-are.html
.. _`hard and soft parses`: http://www.dba-oracle.com/t_hard_vs_soft_parse_parsing.htm
.. _`same plan`: http://stackoverflow.com/a/16012239
.. _`estimated resource usage`: http://docs.oracle.com/cd/E16655_01/server.121/e15858/tgsql_optcncpt.htm#TGSQL195
.. _`query plan`: http://docs.oracle.com/cd/E16655_01/server.121/e15858/tgsql_sqlproc.htm#TGSQL184
.. _`Oracle Technology Network`: http://www.oracle.com/technetwork/documentation/index.html#database



.. toctree::
   :hidden:

   Explain Plan <explain-plan>
   Adaptive Query Optimization <adaptive-query-optimization>