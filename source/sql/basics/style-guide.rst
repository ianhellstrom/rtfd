.. _sql-basics-style:

Style Guide
===========
Before we talk about the optimization of actual SQL queries in Oracle, we want to take a moment and discuss a few best practices regarding style.
These recommendations do not improve the performance of your queries in any way, but they may well increase your productivity, especially when it comes to debugging your code.
Other than that, your credibility as a developer might get a slight bump.

Conventions
-----------
Stick to existing rules regarding style, object nomenclature, comments, and documentation as much as possible.
When it comes to object naming, be sure to follow whatever is generally accepted at your organization.
For example, are underscores used (``FIRST_NAME``) instead of spaces or is it common to simply concatenate words (``FIRSTNAME``)?
If there are no rules or guidelines yet, establish them with your team, write them down with plenty of examples so that they are clear to all, publish them where everyone can see them, and stick to your guns.
Although it should be clear, we'll say it anyway: be consistent.

Use the ANSI-standard ``JOIN`` in ``FROM`` clauses rather than the deprecated versions with commas and the ``(+)`` operator for outer joins.
It's deprecated, so leave it be.

Capitalization
--------------
Keywords, reserved words, reserved namespaces and objects (i.e. tables, columns, indexes, …) are by default case-insensitive in Oracle, unless you have surrounded them by double quotes, like so: ``SELECT 42 AS "THE AnsweR" FROM DUAL``. 
It is generally not recommended that you use case-sensitive object names or names with spaces. 
Translation of object names into more human-readable formats is something that should ultimately be handled by an application and not the database. 
Note, however, that strings can be case-sensitive: ``SELECT last_name FROM people WHERE last_name = 'Jones'`` is different from ``SELECT last_name FROM people WHERE last_name = 'jones'``.
  
Semicolons
----------
Sentences end with full stops, SQL statements with semicolons.
Not all RDBMS clients require a semicolon to execute a single SQL statement, but you save yourself a lot of trouble if you just learn to finish each statement with a semicolon.

Asterisks
---------
Never use ``SELECT *`` in production code.
At some point, someone will come and modify the table or view you're querying from.
If, on the one hand, the column you need in your application has been removed, you'll end up with an application that displays an error.
Best case: you're alerted by an automated unit test that fails, so you branch off and fix the issue before merging it back into the main repository.
Worst case: your client calls you and says that the application displays a runtime error, although the feedback is usually more along the lines of 'It does not work'.
If, on the other hand, several columns have been added you grab more than you actually need, thereby generating unnecessary overhead in the database and additional network traffic, which bring us to the next point:

Thrift
------

Grab only the data you really need.
If a table has a hundred columns and you only need three of them, do not select everything 'just in case'.
You don't go to a car dealer and buy two cars just in case the first one breaks down, do you?
Take what you need: no more, no less.

The same goes for subqueries: if you reuse a subquery multiple times in a larger query, don't copy-paste it.
Instead use a subquery factor or common table expression (i.e. ``WITH`` clause).
It makes your code easier to read, you don't have to update your subquery in several places if you ever need to make changes, and more importantly, Oracle can avoid doing the same thing multiple times.
Oracle sometimes caches a subquery that appears repeatedly in your query, but there is no guarantee.

Factor your code in general.
Portions of stored procedures (or user-defined functions) that you use frequently should become their own stored procedures (or functions).
Whenever a (small) portion of a procedure or function needs to be modified, factored code can minimize the recompilation.
Just because you are working on a database does not mean you can ignore good code design altogether.

When your result set needs to be sorted, use the ``ORDER BY`` clause, but do not force Oracle to sort data when you do not require it to be so.
Oracle generally ignores irrelevant ``ORDER BY`` clauses in subqueries, but it's sloppy to leave them in your code, and it can have an adverse effect on performance in case Oracle does not decide to ignore it.
Moreover, views with ``ORDER BY`` clauses cause multiple sorts to be performed whenever someone selects data from the view but in a different order.

Don't go nuts with minimalism though.
Never use ordinals (a.k.a. the column position) to sort data in production code.
Specify the column names (or aliases) in the ``ORDER BY`` clause and you won't have any problems when someone alters the code.
The same applies to the column list in ``INSERT`` statements; never ever assume that the order in which the columns are provided matches the table you are adding data to, even though the data types happen to match, and that the order of both the source and the target will always stay the same.

Aliases
-------
When you are dealing with more than one table (or view), use *meaningful* aliases.
It reduces the amount of typing and it makes reading the query easier on the eyes.
The adjective meaningful is there to remind you that ``x`` and ``y`` are probably not that revealing, and they do no nothing to aid the legibility of your code.
Moreover, when defining column aliases, use ``AS``.
Its use is optional but sometimes it can be hard to figure out whether you missed a comma between two column names or whether the alias for one column is supposed to be the name of another.

Comments
--------
Add meaningful comments to your code: either use ``/* ... */`` for (multiline) comment blocks or ``--`` for comments that do not extend to the next line.
The key word here is *meaningful*.
Trivial comments should not be added as they clutter your code and are immediately obvious to all but the brain-dead.

Add meaningful comments to the data dictionary with the ``COMMENT`` statement.
You can add comments to tables, (materialized) views, columns, operators and index types.

.. note::
   You can automatically generate documentation (HTML, PDF, CHM, …) from the metadata in the data dictionary (``SELECT * FROM dictionary``) with for instance the option to 'Generate DB Doc' from the connections window/tab in Oracle SQL Developer, Quest Toad's 'HTML Schema Doc Generator' in the Database > Report menu. 
   Specialized tools to extract and display metadata from Oracle's data dictionary exist too: for example, devart's `dbForge Documenter`_ or the free `SchemaSpy`_.
    
Constraints
-----------
We've said it before and we are going to say it again: be consistent.
Especially when it comes to constraints that force user data into straitjackets.
Constraints are imperative to databases.
However, when you add ``NOT NULL`` constraints to columns that can have missing data (``NULL``), you force users to enter rubbish.
As they will soon find out after receiving an error message: a blank space will often do the trick.
Before you think about adding ``TRIM(...)`` or ``REGEXP_LIKE(...)`` checks to all data entered manually, think again: users will also quickly figure out that any random character (combination) will work and you cannot account for all possible situations.
Prior to 11g you may have needed to convert ``NULL`` to ``'N/A'`` or something similar to allow indexing on missing values, but that is not necessary `any longer`_.
The link shows a function-based B-tree index that includes columns with ``NULL``.
By the way, bitmap indexes include rows with ``NULL``; the default index is a B-tree index though.

Respect
-------
No, you don't have to get all Aretha Franklin over your database, but you have to respect data types.
Never rely on implicit data type conversions, and always convince yourself that the data type you think applies, really does apply.
With a simple ``DESC tab_name`` you can remove all doubt.

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

Formatting
----------
Format your SQL queries and format them consistently.
Better yet, use either a built-in formatter or use an `online formatter`_.
Make sure you use the same formatting rules as your colleagues: it helps making sharing and analysing each other's code so much easier.
It may come as a surprise but the actual format matters, even spaces!
The result set that Oracle fetches for you does not depend on spaces but whether it needs to parse a statement with a single space extra.
We shall talk more about (hard/soft) parsing of statements later when we discuss :ref:`execution plans <sql-exec-plan>`, but for now suffice to say that each query needs to be hashed and analysed by Oracle before it can execute it.
If the query hashes are the same, which generally means that the query you have submitted is formatted identically as one in memory (the system global area (:term:`SGA`) to be precise), Oracle can immediately execute it. If not, Oracle needs to analyse your query first.
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

Coding Guidelines
-----------------
We recommend that each organization define a programming standards document that clearly specifies how to write consistent and maintainable code.
At the very least the coding standards should tell you how to name objects and format code.
That includes, but is not limited to, standard prefixes for all database objects, notation standards (e.g. keywords in upper case, application-specific identifiers in lower case, underscores between words in identifiers), maximum line length, line break rules, indentation spaces for code blocks, and default headers.
If your IDE supports IntelliSense or something similar, then  `Hungarian notation`_ may be overkill, but for complex programs it may be beneficial to prefix the logical (Apps Hungarian) or the physical (Systems Hungarian) type to avoid collisions, although the former is often to be preferred to the latter.

An example of a comprehensive set of coding guidelines for both SQL and PL/SQL is by `Ian Hellström`_.
The document's source is in Markdown and `publicly available`_ in order to make it easy for you to adapt it to your (organization's) needs.
`Steven Feuerstein's`_  and `topcoder's`_ best practices and programming standards focus mainly on PL/SQL.

.. _`dbForge Documenter`: https://www.devart.com/dbforge/oracle/documenter/
.. _`SchemaSpy`: http://schemaspy.sourceforge.net/
.. _`online formatter`: http://www.dpriver.com/pp/sqlformat.htm
.. _`DBA Oracle`: http://www.dba-oracle.com/t_sql_statements_formatting.htm
.. _`any longer`: http://www.dba-oracle.com/oracle_tips_null_idx.htm
.. _`Hungarian notation`: http://programmers.stackexchange.com/a/39874
.. _`Steven Feuerstein's`: http://www.toadworld.com/platforms/oracle/w/wiki/8245.plsql-standards.aspx
.. _`topcoder`: http://www.topcoder.com/i/development/uml/Oracle_PLSQL_Coding_Guidelines.pdf
.. _`Ian Hellström`: https://ianhellstrom.org/guidelines.html
.. _`publicly available`: https://github.com/ianhellstrom/html-docs
.. _`Steven Feuerstein's`: http://www.toadworld.com/platforms/oracle/w/wiki/8245.plsql-standards.aspx
.. _`topcoder's`: http://www.topcoder.com/i/development/uml/Oracle_PLSQL_Coding_Guidelines.pdf
