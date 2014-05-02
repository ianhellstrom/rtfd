.. _sql-intro:

************
What is SQL?
************
Before Edgar F. Codd formulated the relational model [Codd69]_ [Codd70]_ for database management in 1969/1970 we had the Dark Ages of databases.
Applications that required some form of stored data used a database unique to the application.
Therefore, each development project had to reinvent the wheel again.

In the 1970s Donald Chamberlin and Raymond Boyce developed the Structured English Query Language (SEQUEL) at IBM to work with data stored in System R, IBM's database management system of the time.
Because of trademark dispute they later changed the name to SQL, which stands for Structured Query Language.
Because of that there are quite a few people who still pronounce SQL as 'sequel', whereas others say 'S-Q-L'.
If you search the internet you are likely to find an equal amount of proponents of either school of `pronunciation`_.
Honestly, who gives a hoot!

While IBM were still tinkering with their prototype System R, Larry Ellison, Bob Miner and Ed Oates founded Software Development Laboratories (SDL) in 1977.
The first version (1978) was never released officially but the code name 'Oracle', taken from the CIA database project all three founders had worked on while at Ampex Corporation, remains to this day.
Another years later they - now calling themselves  Relational Software Inc. (RSI) - released Oracle V2, the first commercially available implementation of SQL.
It was not until 1979 that IBM had a commercial database product; IBM DB2 came out in 1983.
A year before the release of DB2, Oracle decided they had enough of RSI and they became known as Oracle Systems Corporation, named after their primary product.
In 1995 Oracle Systems Corporation was rechristened Oracle Corporation.

Oracle is still the leader when it comes to relational database management software, although the competition is getting stronger (especially on Windows platforms where Microsoft is in the lead with SQL Server) and more diverse: NoSQL (`Cassandra`_, `MongoDB`_, `Neo4j`_, …), NewSQL (`Clustrix`_, `H-Store`_), in-memory databases (IMDB) like SAP's `HANA`_, highly distributed systems for cloud computing like `Apache Hadoop`_, and so on.

A nice indicator of database popularity is `DB engines`_.
Their ranking does not compare to in-depth market research by `Gartner`_ and the likes but of course it's free and not that far from the truth.

Because of its ubiquity SQL became a standard of ANSI in 1986 and ISO one year later.
Before you start shouting 'Halleluja!' from the rooftops, bear in mind that large parts of SQL are the same across the many database vendors but many are not.
For example:

* ``COALESCE(...)`` is the ANSI SQL function that Oracle has implemented but most Oracle SQL developers use ``NVL(...)`` instead; in SQL Server there is ``ISNULL(...)`` even though ``COALESCE(...)`` works too.
* ``SELECT 3.1415 AS pi`` gives you a table with one row and one column in SQL Server, but all Oracle gives you is an ``ORA-00923`` error because it did not find the ``FROM`` keyword where it expected it.
  Oracle needs ``FROM DUAL`` to make the query work.
* Returning ten rows from a result set can be accomplished with ``SELECT TOP 10`` in SQL Server.
  Oracle requires you apply a filter like ``WHERE ROWNUM <= 10``. 
  To get the actual top-10 rows (based on some ordering), Oracle requires a subquery with an ``ORDER BY`` clause, whereas SQL Server allows you to simply issue ``ORDER BY`` in the same query. 
  As of `12c`_ it is possible to use the `row limiting clause`_ though: ``FETCH FIRST 10 ROWS ONLY``, which comes after the ``ORDER BY`` clause.
* Window functions (i.e. aggregate functions in the ``SELECT`` clause that are accompanied by an ``OVER`` clause with ``PARTITION BY`` and/or ``ORDER BY``) are another cause of portability headaches: the `SQL:2011 <http://www.iso.org/iso/search.htm?qt=ISO+9075&sort_by=rel&type=simple&published=on&active_tab=standards>`_ (ISO/IEC 9075:2011) standard defines a window clause (``WINDOW``) that enables easy reuse (through an alias) of the same window but as of this writing no major RDBMS vendor, save for the open-source PostgreSQL project, has implemented the window clause.
* Hierarchical queries are done with the ``CONNECT BY`` clause in Oracle, whereas SQL Server requires recursive common table expressions.
  Only since 11g R2 does Oracle support recursive common table expressions.
* There are three inequality comparison operators in Oracle: ``<>``, which is the standard operator, ``!=``, which most databases accept too, and ``^=``, which happens to be supported by IBM DB2 but not by Microsoft SQL Server, for instance.

We could go on with our list but you probably get the message: even if you stick to the ANSI SQL standard as much as possible, you may not end up with portable code.
Anyway, portability is by no means necessary: if you spend all your days working with Oracle databases, why care about ``NVL(...)`` not being understood by SQL Server? `Joe Celko's books`_ are great if you are interested in ANSI-compliant SQL tips and tricks.

As we have said before, we shall assume that you have knowledge of and considerable experience with SQL and Oracle.
If not, you can read on but we do not recommend it.

.. _relational model: http://en.wikipedia.org/wiki/Relational_model
.. _pronunciation: http://patorjk.com/blog/2012/01/26/pronouncing-sql-s-q-l-or-sequel
.. _Cassandra: http://cassandra.apache.org
.. _MongoDB: http://www.mongodb.org
.. _Neo4j: http://www.neo4j.org
.. _Clustrix: http://www.clustrix.com
.. _H-Store: http://hstore.cs.brown.edu
.. _HANA: http://www.sap.com/pc/tech/in-memory-computing-hana.html
.. _Apache Hadoop: http://hadoop.apache.org
.. _DB engines: http://db-engines.com/en/ranking
.. _Gartner: http://www.gartner.com/technology/reprints.do?id=1-1M9YEHW&ct=131028&st=sb
.. _12c: http://www.oracle-base.com/articles/12c/row-limiting-clause-for-top-n-queries-12cr1.php#top-n
.. _row limiting clause: http://docs.oracle.com/cd/E16655_01/server.121/e17209/statements_10002.htm#SQLRF55631
.. _Joe Celko's books: http://www.amazon.com/Joe-Celko/e/B000ARBFVQ

.. [Codd69] *Derivability, Redundancy, and Consistency of Relations Stored in Large Data Banks*, E.F. Codd, IBM Research Report, 1969.
.. [Codd70] *A relational model of data for large shared data banks*, E.F. Codd, Communication of the ACM, Vol. 13, pp. 377—387, 1970. DOI: `10.1145/362384.362685 <http://dx.doi.org/10.1145%2F362384.362685>`_.
