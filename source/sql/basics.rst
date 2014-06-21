.. _sql-basics:

**********
SQL Basics
**********

RDBMS basics
============
Relational database management systems, such as Oracle, are built on two pillars of mathematics: set theory and (first-order) predicate logic.
In a database live objects and these objects have certain relations to one another.
The objects have properties that are `quantifiable`_ and we can use these properties to compare objects.

All data is represented as *n*-ary relations.
Each relation consists of both a heading and a body.
A heading is a set of attributes, and a body of an *n*-ary relation is a set of *n*-tuples with no specific order of its elements.
A sets is an *unordered* collection of *unique* elements: the sets {a,b,c} and {b,c,a,c,c} are equivalent.
Whereas mathematicians generally use two-valued logic to reason about about data, databases use three-valued logic: true, false, and unknown (``NULL``).

Note that up to this point we have not talked about tables or columns at all.
So far we have been looking at the high-level conceptual database model, which consists only of entities (e.g. Location, Department, Product, Supplier, Customer, Sales, …) and relations.
The conceptual data model describes *what* we want to model.
When we move on to the logical data model, we need to add attributes and primary keys to our entities and relations.
We are still independent of the particulars of a database management system but we have to define *how* we want to model our entities and relations.
Common logical data models include the relational, network, hierarchical, flat, entity-relationship, and object-relational model.
Since Oracle is a relational database management system (RDBMS) we shall focus on that one here.

Once we know what we want to model and how we intend to model our high-level entities and relations, we must *specify* our logical data model, that is, we define our physical data model, which is highly dependent on the RDBMS we use: tables, views, columns, data types, constraints, indexes, procedures, roles, and so on.
Attributes are represented by columns, tuples by rows and relations by tables.
A nice, brief overview of the three levels of data models is available on `1Keydata`_.

With the risk of sounding pedantic, we wish to emphasize that tables are logical beasts: they have logical rows and columns.
Records and fields are their physical equivalents; fields are housed in the user interfaces of client applications, and records hang out in files and cursors.
We shall try and not confuse them but we don't want to make promises we can't keep.

Important to note is that rows can appear more than once in relational databases, so the idea that we can have only distinct elements in sets does not strictly apply; with the ``DISTINCT`` clause you can again obtain all unique elements but that is not really the point.
`Multisets`_ provide the appropriate generalization upon which RDBMSs are actually based, but even then SQL will deviate from the relational model.
For instance, columns can be anonymous.
Yes, we know: Oracle automatically assigns the expression of a column without an alias as the column name when outputting but that does not make it an actual attribute — try accessing it from outside a subquery or CTAS'ing into a new table without an ``ORA-00998`` error telling you to ``name this expression with a column alias``.
Anyway, we shall not dwell on any additional idiosyncrasies pertaining to the relational model.
In case you do crave for more details on the relational model though, we recommend the book *SQL and Relational Theory* by `Christopher J. Date`_.

.. _quantifiable: http://en.wikipedia.org/wiki/Quantification#Logic
.. _1Keydata: http://www.1keydata.com/datawarehousing/data-modeling-levels.html
.. _Christopher J. Date: http://www.amazon.com/SQL-Relational-Theory-Write-Accurate/dp/1449316409/
.. _Multisets: http://en.wikipedia.org/wiki/Multiset

.. _sql-style:

Style Guide
===========
Before we talk about the optimization of actual SQL queries in Oracle, we want to take a moment and discuss a few best practices regarding style.
These recommendations do not improve the performance of your queries in any way, but they may well increase your productivity, especially when it comes to debugging your code.
Other than that, your credibility as a developer might get a slight bump.

* **Conventions**.

  * Stick to existing rules regarding style, object nomenclature, comments, and documentation as much as possible.
    When it comes to object naming, be sure to follow whatever is generally accepted at your organization.
    For example, are underscores used (``FIRST_NAME``) instead of spaces or is it common to simply concatenate words (``FIRSTNAME``)?
    If there are no rules or guidelines yet, establish them with your team, write them down with plenty of examples so that they are clear to all, publish them where everyone can see them, and stick to your guns.
    Although it should be clear, we'll say it anyway: be consistent.

  * Use the ANSI-standard ``JOIN`` in ``FROM`` clauses rather than the deprecated versions with commas and the ``(+)`` operator for outer joins.
    It's deprecated, so leave it be.

* **Capitalization**. 
  Keywords, reserved words, reserved namespaces and objects (i.e. tables, columns, indexes, …) are by default case-insensitive in Oracle, unless you have surrounded them by double quotes, like so: ``SELECT 42 AS "THE AnsweR" FROM DUAL``. 
  It is generally not recommended that you use case-sensitive object names or names with spaces. 
  Translation of object names into more human-readable formats is something that should ultimately be handled by an application and not the database. 
  Note, however, that strings can be case-sensitive: ``SELECT last_name FROM people WHERE last_name = 'Jones'`` is different from ``SELECT last_name FROM people WHERE last_name = 'jones'``.
  
* **Semicolons**.
  Sentences end with full stops, SQL statements with semicolons.
  Not all RDBMS clients require a semicolon to execute a single SQL statement, but you save yourself a lot of trouble if you just learn to finish each statement with a semicolon.

* **Asterisks**.
  Never use ``SELECT *`` in production code.
  At some point, someone will come and modify the table or view you're querying from.
  If, on the one hand, the column you need in your application has been removed, you'll end up with an application that displays an error.
  Best case: you're alerted by an automated unit test that fails, so you branch off and fix the issue before merging it back into the main repository.
  Worst case: your client calls you and says that the application displays a runtime error, although the feedback is usually more along the lines of 'It does not work'.
  If, on the other hand, several columns have been added you grab more than you actually need, thereby generating unnecessary overhead in the database and additional network traffic, which bring us to the next point:

* **Thrift**.

  * Grab only the data you really need.
    If a table has a hundred columns and you only need three of them, do not select everything 'just in case'.
    You don't go to a car dealer and buy two cars just in case the first one breaks down, do you?
    Take what you need: no more, no less.

  * The same goes for subqueries: if you reuse a subquery multiple times in a larger query, don't copy-paste it.
    Instead use a subquery factor or common table expression (i.e. ``WITH`` clause).
    It makes your code easier to read, you don't have to update your subquery in several places if you ever need to make changes, and more importantly, Oracle can avoid doing the same thing multiple times.
    Oracle sometimes caches a subquery that appears repeatedly in your query, but there is no guarantee.

  * Factor your code in general.
    Portions of stored procedures (or user-defined functions) that you use frequently should become their own stored procedures (or functions).
    Whenever a (small) portion of a procedure or function needs to be modified, factored code can minimize the recompilation.
    Just because you are working on a database does not mean you can ignore good code design altogether.

  * When your result set needs to be sorted, use the ``ORDER BY`` clause, but do not force Oracle to sort data when you do not require it to be so.
    Oracle generally ignores irrelevant ``ORDER BY`` clauses in subqueries, but it's sloppy to leave them in your code, and it can have an adverse effect on performance in case Oracle does not decide to ignore it.
    Moreover, views with ``ORDER BY`` clauses cause multiple sorts to be performed whenever someone selects data from the view but in a different order.

  * Don't go nuts with minimalism though.
    Never use ordinals (a.k.a. the column position) to sort data in production code.
    Specify the column names (or aliases) in the ``ORDER BY`` clause and you won't have any problems when someone alters the code.
    The same applies to the column list in ``INSERT`` statements; never ever assume that the order in which the columns are provided matches the table you are adding data to, even though the data types happen to match, and that the order of both the source and the target will always stay the same.

* **Aliases**.
  When you are dealing with more than one table (or view), use *meaningful* aliases.
  It reduces the amount of typing and it makes reading the query easier on the eyes.
  The adjective meaningful is there to remind you that ``x`` and ``y`` are probably not that revealing, and they do no nothing to aid the legibility of your code.
  Moreover, when defining column aliases, use ``AS``.
  Its use is optional but sometimes it can be hard to figure out whether you missed a comma between two column names or whether the alias for one column is supposed to be the name of another.

* **Comments**.

  * Add meaningful comments to your code: either use ``/* ... */`` for (multiline) comment blocks or ``--`` for comments that do not extend to the next line.
    The key word here is *meaningful*.
    Trivial comments should not be added as they clutter your code and are immediately obvious to all but the brain-dead.

  * Add meaningful comments to the data dictionary with the ``COMMENT`` statement.
    You can add comments to tables, (materialized) views, columns, operators and index types.
    Note that you can automatically generate documentation (HTML, PDF, CHM, …) from the metadata in the data dictionary (``SELECT * FROM dictionary``) with for instance the option to 'Generate DB Doc' from the connections window/tab in Oracle SQL Developer, Quest Toad's 'HTML Schema Doc Generator' in the Database > Report menu. Specialized tools to extract and display metadata from Oracle's data dictionary exist too: for example, the xSQL's excellent `Database Documenter`_ or the free `SchemaSpy`_.
    
* **Constraints**.
  We've said it before and we are going to say it again: be consistent.
  Especially when it comes to constraints that force user data into straitjackets.
  Constraints are imperative to databases.
  However, when you add ``NOT NULL`` constraints to columns that can have missing data (``NULL``), you force users to enter rubbish.
  As they will soon find out after receiving an error message: a blank space will often do the trick.
  Before you think about adding ``TRIM(...)`` or ``REGEXP_LIKE(...)`` checks to all data entered manually, think again: users will also quickly figure out that any random character (combination) will work and you cannot account for all possible situations.
  Prior to 11g you may have needed to convert ``NULL`` to ``'N/A'`` or something similar to allow indexing on missing values, but that is not necessary `any longer`_.
  The link shows a function-based B-tree index that includes columns with ``NULL``.
  By the way, bitmap indexes include rows with ``NULL``; the default index is a B-tree index though.

* **Respect**.
  No, you don't have to get all Aretha Franklin over your database, but you have to respect data types.
  Never rely on implicit data type conversions, and always convince yourself that the data type you think applies, really does apply.
  With a simple ``DESC table_name`` you can remove all doubt.
  
  If you're not convinced, please take a look at the following example, which shows you what you get when you sort numerical-looking data that is actually stored as a string.

  .. code-block:: sql
     :linenos:
     
     WITH
       raw_data AS
       (
         SELECT 1 AS int_as_number, '1' AS int_as_varchar FROM dual
         UNION ALL
         SELECT 2 AS int_as_number, '2' AS int_as_varchar FROM dual
         UNION ALL
         SELECT 3 AS int_as_number, '3' AS int_as_varchar FROM dual
         UNION ALL
         SELECT 12 AS int_as_number, '12' AS int_as_varchar FROM dual
         UNION ALL
         SELECT 28 AS int_as_number, '28' AS int_as_varchar FROM dual
       )
     SELECT * FROM raw_data ORDER BY int_as_varchar;

  The moral: do not assume anything when it comes to data types. Just because something looks like a number does not mean that it is stored as a number.

* **Formatting**.
  Format your SQL queries and format them consistently.
  Better yet, use either a built-in formatter or use an `online formatter`_.
  Make sure you use the same formatting rules as your colleagues: it helps making sharing and analysing each other's code so much easier.
  It may come as a surprise but the actual format matters, even spaces!
  The result set that Oracle fetches for you does not depend on spaces but whether it needs to parse a statement with a single space extra.
  We shall talk more about (hard/soft) parsing of statements later when we discuss :ref:`execution plans <sql-exec-plan>`, but for now suffice to say that each query needs to be hashed and analysed by Oracle before it can execute it.
  If the query hashes are the same, which generally means that the query you have submitted is formatted identically as one in memory (the system global area (SGA) to be precise), Oracle can immediately execute it. If not, Oracle needs to analyse your query first.
  As said on `DBA Oracle`_, the time Oracle needs to parse a statement is almost negligible, but when many users issue functionally and syntactically identical yet symbolically distinct statements, the small amounts of time can quickly add up.

Although there is no general consensus about good formatting rules, you can add line breaks in appropriate places, so you are able to comment or uncomment lines without having to manually reformat your code every time. This is particularly useful when you are debugging more complex queries. To do so, insert line breaks

* before and after ``SELECT``, ``INSERT``, ``UPDATE``, ``DELETE``, ``FROM``, ``JOIN``, ``ON`` ``WHERE``, ``CONNECT BY``, ``START WITH``, ``GROUP BY``, ``HAVING``, and ``ORDER BY``
* before and after ``DECLARE``, ``BEGIN``, ``END``, ``LOOP``, ``EXCEPTION`` in PL/SQL blocks
* after ``AS`` or ``IS`` in ``CREATE`` statements
* before ``WHEN``, ``ELSE``, and ``END`` in ``CASE`` statements
* before ``AND`` and ``OR``
* before commas
* before semicolons
* after the first, and before the last bracket of a large expression.

Moreover, we recommend that each organization define a programming standards document that clearly specifies how to write consistent and maintainable code.
At the very least the coding standards should tell you how to name objects and format code.
That includes, but is not limited to, standard prefixes for all database objects, notation standards (e.g. keywords in upper case, application-specific identifiers in lower case, underscores between words in identifiers), maximum line length, line break rules, indentation spaces for code blocks, and default headers.
If your IDE supports IntelliSense or something similar, then  `Hungarian notation`_ may be overkill, but for complex programs it may be beneficial to prefix the logical (Apps Hungarian) or the physical (Systems Hungarian) type to avoid collisions, although the former is often to be preferred to the latter.
Two examples of programming standards documents for PL/SQL are `Steven Feuerstein's`_  or the one on `topcoder`_.

.. _Database Documenter: http://www.xsql.com/products/database_documenter/
.. _SchemaSpy: http://schemaspy.sourceforge.net/
.. _online formatter: http://www.dpriver.com/pp/sqlformat.htm
.. _DBA Oracle: http://www.dba-oracle.com/t_sql_statements_formatting.htm
.. _any longer: http://www.dba-oracle.com/oracle_tips_null_idx.htm
.. _Hungarian notation: http://programmers.stackexchange.com/a/39874
.. _Steven Feuerstein's: http://www.toadworld.com/platforms/oracle/w/wiki/8245.plsql-standards.aspx
.. _topcoder: http://www.topcoder.com/i/development/uml/Oracle_PLSQL_Coding_Guidelines.pdf


.. _sql-proc-order:

Query Processing Order
======================
Important to understand before we discuss execution plans is how Oracle processes queries logically.
Let's look at the following query:

.. code-block:: sql
   :linenos:
   
   SELECT 
               f.product AS beer
             , p.product AS crisps
   FROM        fridge f 
   CROSS JOIN  pantry p 
   WHERE       f.product         = 'Beer'
               AND f.temperature < 5 
               AND f.size        = '50 fl oz' 
               AND p.product     = 'Crisps'
               AND p.style       = 'Cream Cheese' 
               AND p.size        = '250g'
   ORDER BY    crisps
             , beer
   ;

What does this query tell you other than that you're a tad peckish, extremely thirsty, and that the fridge and pantry seem to use different systems of measurement?

You may think that it reads the query in the way that we type it, but Oracle (and other RDBMSs too) does not read from top to bottom.
It more or less reads our queries upside down.
Not exactly, but we'll see what it does in a minute.
Oracle is a fancy machine that translates our SQL statements into something it can understand and execute.
In essence, it's a data robot that does exactly what we tell it to do.
Now, suppose you have purchased a robot to help you around the house, and its first and foremost task is to assist you in quenching your thirst and sating your appetite.
How would you tell it go fetch a beer and a packet of crisps?

Well, you'd probably tell it to go to the fridge, look for beer, grab a bottle (50 fl oz) with a temperature below 5 degrees Celsius, then go to the pantry and look for a 250g packet of cream cheese crisps. Once it's done, it should come back to you and place the items in front of you, sorted in the way you asked it to do.
That's right, you first tell it to go to the place where the fridge and the pantry are located (probably the kitchen: ``FROM``), then to look for everything that matches your criteria (``WHERE``), and finally to return the items (``SELECT``) sorted in the order you specified (``ORDER BY``).

That's pretty much what Oracle does too. 
The order in which clauses are logically processed by Oracle is as follows: ``FROM -> CONNECT BY -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY``.

Of course, your query does not have to have every clause, and some cannot even be used with/without others (e.g. ``HAVING`` can only be used when you use ``GROUP BY``).

.. _fig-proc-order:

.. figure:: images/query-proc-order.*
   :scale: 60%
   :alt: query processing order
   
   Oracle's query processing order, including optional clauses.

The processing order is also the reason why the previous query worked like a charm and the following will result in an error:

.. code-block:: sql
   :linenos:
   
   SELECT 
             product          AS item
           , MIN(temperature) AS min_temperature
           , COUNT(*)         AS num_products
   FROM      fridge
   GROUP BY  item
   ;         

When Oracle processes the ``GROUP BY`` clause the alias ``item`` is not yet known, so you are greeted by an ``ORA-00904: invalid identifier`` error.

.. _sql-exec-plan:

Execution Plans
===============
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
To make life easier it is often best to add a comment unique to your statement, for instance ``SELECT /* my_custom_comment */ last_name, first_name FROM people``.
Then you can simply look for your query from ``V$SQL``:

.. code-block:: sql
   :linenos:
   
   SELECT 
           sql_id
         , hash_value
         , plan_hash_value
         , sql_text
   FROM    v$sql
   WHERE   sql_text LIKE 'SELECT /* my_custom_comment */%'
   ;

In case you happen to know the SQL ID already and would like to know the corresponding hash value, you can use the function ``DBMS_UTILITY.SQLID_TO_SQLHASH``, which takes the ``sql_id`` (``VARCHAR2``) and returns a ``NUMBER``. 
Note that *all* characters affect the hash value, including spaces and line breaks as well as capitalization and of course comments.

The last stage of the parser is to look for possible shortcuts by sifting through the shared pool, which is a *portion of the SGA that contains shared memory constructs such as shared SQL areas*, which hold the parse tree and execution plans for SQL statements; each unique statement has only one shared SQL area.
We can distinguish two cases: `hard and soft parses`_.

#. **Soft parse** (library cache hit): if the statement hashes to a value that is identical to one already present in the shared pool *and* the texts of the matching hash values are the same *and* its parsed representation can be shared, Oracle looks up the execution plan and executes the statement accordingly.
   Literals must also be the same for Oracle to be able to use the same shared SQL area; the exception is when ``CURSOR_SHARING`` is set to ``FORCE``.

#. **Hard parse** (library cache miss): if the statement has a hash value different from the ones that are available in the SGA *or* its parsed representation cannot be shared, Oracle hands the code over to the query optimizer.
   The query optimizer then has to build an executable version from scratch.

Criteria for when a SQL statement or PL/SQL block can be shared are described in the *Oracle Database Performance Tuning Guide*, which can be found `here <http://docs.oracle.com/cd/E16655_01/server.121/e15857/tune_shared_pool.htm#TGDBA564>`_. 
Basically, the statements' hashes and texts, all referenced objects, any bind variables (name, data type, and length), and the session environments have to match. PL/SQL blocks that do not use bind variables are said to be not re-entrant, and they are always hard-parsed.
To find out why statements cannot be shared you can use the view ``V$SQL_SHARED_CURSOR``.

Perhaps you noticed that we had sneaked in the column ``plan_hash_value`` in the ``V$SQL`` query above. 
SQL statements with different hash values can obviously have the `same plan`_, which means that their plan hash values are equal.
Please note that the plan hash value is merely an `indicator`_ of similar operations on database objects: filter and access predicates, which we shall discuss in more detail, are not part of the plan hash value calculation.

For hard parses, the next station on the SQL compiler line is the query optimizer.
The query optimizer, or just optimizer, is the *built-in database software that determines the most efficient way to execute a SQL statement*. 
The optimizer is also known as the cost-based optimizer (CBO), and it consists of the query transformer, the estimator, and the plan generator:

* The query transformer *decides whether to rewrite a user query to generate a better query plan, merges views, and performs subquery unnesting*.
* The estimator *uses statistics [from the data dictionary] to estimate the selectivity, cardinality, and cost of execution plans. 
  The main goal of the estimator is to estimate the overall cost of an execution plan*.
* The plan generator *tries out different possible plans for a given query so that the query optimizer can choose the plan with the lowest cost. It explores different plans for a query block by trying out different access paths, join methods, and join orders*. The optimizer also evaluates expressions, and it can convert correlated subqueries into equivalent join statements or vice versa.

What the optimizer does, in a nutshell, is apply fancy heuristics to figure out the best way to execute your query: it calculates alternate routes from your screen through the database back to your screen, and picks the best one.
By default Oracle tries to minimize the `estimated resource usage`_ (i.e. maximize the throughput), which depends on I/O, CPU, memory, the number of rows returned, and the size of the initial data sets.
The objective of the optimization can be altered by changing the value of ``OPTIMIZER_MODE`` parameter.

If you can recall our :ref:`example <sql-proc-order>` of the robot, the beer, and the packet of crisps, you may remember that the robot had to check both the pantry and the fridge.
If we equip our robot with Oracle's query optimizer, the robot will not simply walk to the kitchen, find the items by searching for them, and then return with our refreshments, but try to do it as efficiently as possible.
It will modify our query without altering the query's function (i.e. fetch you some booze and a few nibbly bits), and explore its options when it comes to retrieving the items from the kitchen.
For instance, if we happen to have a smart fridge with a display on the door that shows where all bottles are located, including the contents, temperature, and size of each bottle, the robot does not have to rummage through decaying fruit and vegetables at the bottom in the hope that a bottle is located somewhere underneath the rubbish.
Instead it can look up the drinks (from an index) and fetch the bottles we want by removing them the spots highlighted on the display (by ROWID).
What the optimizer can also figure out is whether it is more advantageous to grab the beers and then check the pantry, or the other way round (join order).
If the robot has to go down a few steps to obtain crisps while still holding the beer in its hands, the optimizer may decide that carrying the heavy and/or many beverages may be inefficient.
Furthermore, it may decide to pick up a tray and place the products it has already extracted on it (temporary table) while continuing its search for the remaining items.
Because the optimizer evaluates expressions it will know whether or not it has to do something: a predicate like ``0 = 1`` will be immediately understood by the optimizer, so that our friendly robot knows it has to do bugger all beforehand.

After the optimizer has given its blessings to the optimal execution plan, the row source generator is let loose on that plan. The row source generator produces an iterative plan, which is known as the `query plan`_. The query plan is a binary program that produces the result set when executed. 
It is structured as a row source tree, where a row source is the combination of a set of rows returned by a step in the execution plan and *a control structure that can iteratively process the rows*, that is one row at a time. The row source tree shows an ordering of the tables, an access method for each table, a join method for tables affected by join operations, and data operations (filter, sort, aggregation)

During execution, the SQL engine executes each row source in the tree produced by the row source generator. This step is the only mandatory step in DML processing.

.. _database version: http://www.oracle.com/technetwork/documentation/index.html#database
.. _few hundred characters: http://www.dba-oracle.com/concepts/hashing.htm
.. _indicator: http://oracle-randolf.blogspot.de/2009/07/planhashvalue-how-equal-and-stable-are.html
.. _hard and soft parses: http://www.dba-oracle.com/t_hard_vs_soft_parse_parsing.htm
.. _same plan: http://stackoverflow.com/a/16012239
.. _estimated resource usage: http://docs.oracle.com/cd/E16655_01/server.121/e15858/tgsql_optcncpt.htm#TGSQL195
.. _query plan: http://docs.oracle.com/cd/E16655_01/server.121/e15858/tgsql_sqlproc.htm#TGSQL184