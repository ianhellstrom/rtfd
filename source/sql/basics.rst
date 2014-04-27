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
Yes, we know: Oracle automatically assigns the expression of a column without an alias as the column name when outputting but that does not make it an actual attribute — try accessing it from outside a subquery or CTAS'ing into a new table without an ORA-00998 error telling you to ``name this expression with a column alias``.
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
    If there are no rules or guidelines yet, establish them with your team, write them down with enough of examples so that they are clear to all, publish them where everyone can see them, and stick to your guns.
    Although it should be clear, we'll say it anyway: be consistent.

  * Use the ANSI-standard ``JOIN`` in ``FROM`` clauses rather than the deprecated versions with commas and the ``(+)`` operator for outer joins.
    It's deprecated, so leave it be.

* **Capitalization**. Keywords, reserved words, reserved namespaces and objects (i.e. tables, columns, indexes, …) are by default case-insensitive in Oracle, unless you have surrounded them by double quotes, like so: ``SELECT 42 AS "THE AnsweR" FROM DUAL``. It is generally not recommended that you use case-sensitive object names or names with spaces. Translation of object names into more human-readable formats is something that should ultimately be handled by an application and not the database. Note, however, that strings can be case-sensitive: ``SELECT last_name FROM people WHERE last_name = 'Jones'`` is different from ``SELECT last_name FROM people WHERE last_name = 'jones'``.
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

  * Add meaningful comments to your code: either use ``/* … */`` for (multiline) comment blocks or ``--`` for comments that do not extend to the next line.
    The key word here is *meaningful*.
    Trivial comments should not be added as they clutter your code and are immediately obvious to all but the brain-dead.

  * Add meaningful comments to the data dictionary with the ``COMMENT`` statement.
    You can add comments to tables, (materialized) views, columns, operators and index types.
    Note that you can automatically generate documentation (HTML, PDF, CHM, …) from the metadata in the data dictionary (``SELECT * FROM dictionary``) with for instance the option to 'Generate DB Doc' from the connections window/tab in Oracle SQL Developer, Quest Toad's 'HTML Schema Doc Generator' in the Database > Report menu. Specialized tools to extract and display metadata from Oracle's data dictionary exist too: for example, the xSQL's excellent `Database Documenter`_ or the free `SchemaSpy`_.

* **Formatting**
  Format your SQL queries and format them consistently.
  Better yet, use either a built-in formatter or use an `online formatter`_.
  Make sure you use the same formatting rules as your colleagues: it helps making sharing and analysing each other's code so much easier.
  It may come as a surprise but the actual format matters, even spaces!
  The result set that Oracle fetches for you does not depend on spaces but whether it needs to parse a statement with a single space extra.
  We shall talk more about (hard/soft) parsing of statements later when we discuss execution plans (see :ref:`sql-execplan`), but for now suffice to say that each query needs to be hashed and analysed by Oracle before it can execute it.
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

.. _Database Documenter: http://www.xsql.com/products/database_documenter/
.. _SchemaSpy: http://schemaspy.sourceforge.net/
.. _online formatter: http://www.dpriver.com/pp/sqlformat.htm
.. _DBA Oracle: http://www.dba-oracle.com/t_sql_statements_formatting.htm

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
That's right, you first tell it to go to the place where the fridge and the pantry are located (probably the kitchen: ``FROM``), then to look for everything that matches your criteria (``WHERE``), and finally to return the items sorted in the order you specified (``ORDER BY``).

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

.. _sql-execplan:

Execution Plans
===============
What happens to your SQL statement when you hit execute?