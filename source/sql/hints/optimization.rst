.. _sql-hints-optimization:

SQL Optimization Techniques
===========================
Before you start fidgeting with individual SQL statements, it is important to note that hints are probably the last thing you should consider adding when attempting to optimize your code.
There are several levels of optimization and `it is recommended`_ that you start with the server, then the database instance, and finally go down through the database objects to individual statements.
After all, the effect of any changes made to individual statements, particularly hints, may be lost when the database is fine-tuned later on.

As a database developer/architect you may not want to tread the path that leads to the desk of the DBA.
Fortunately, there is a bunch of things you can do to improve the runtime performance of your statements:

* Optimize access structures:

  * Database design and normalization.
  * Tables: heap or index-organized tables, and table or indexed clusters.
  * Indexes.
  * Constraints.
  * Materialized views.
  * Partitioning schemes.
  * Statistics, including a comprehensive refresh strategy.

* Rewrite SQL statements:

  * Exclude projections that are not required.
  * Minimize the amount of work done more than once.
  * Factor subqueries that are used multiple times in the same statement.
  * Use ``EXISTS`` instead of ``IN`` because the former stops processing once it has found a match.
  * Use ``CASE`` and/or ``DECODE`` to avoid having to scan the same rows over and over again, especially for aggregation functions that act on different subsets of the same data.
  * Use analytic functions to do multiple or moving/rolling aggregations with a single pass through the data.
  * Avoid scalar subqueries in the ``SELECT``-list.
  * Use joins instead of subqueries, as it gives the optimizer more room to play around in.
  * Say what you mean and pick the right join: if you only need an inner join don't write an outer join.
  * Add logically superfluous predicates that may still aid in the search for an optimal execution plan, particularly for outer joins.
  * Avoid implicit conversions of data types, especially in the ``WHERE`` clause.
  * Write ``WHERE`` clause predicates with a close eye on the indexes available, including the leading edge of a composite index.
  * Avoid, whenever possible, comparison operators such as ``<>``, ``NOT IN``, ``NOT EXISTS``, and ``LIKE`` without a leading ``'%'`` for indexed columns in predicates.
  * Do not apply functions on indexed columns in the ``WHERE`` clause when there is no corresponding function-based index.
  * Don't abuse ``HAVING`` to filter rows *before* aggregating.
  * Avoid unnecessary sorts, including when ``UNION ALL`` rather than ``UNION`` is applicable.
  * Avoid ``DISTINCT`` unless you have to use it.
  * Use PL/SQL, especially packages with stored procedures (and bind variables) and shared cursors to provide a clean interface through which all data requests are handled.
  * Add hints once you have determined that it is right and necessary to do so.

The advantage of PL/SQL packages to provide all data to users is that there is, when set up properly, exactly one place where a query is written, and that's the only place where you have to go to to change anything, should you ever wish or need to modify the code.
PL/SQL will be in our sights in the next part but suffice to say it is the key to maintainable code on Oracle.
Obviously, ad-hoc queries cannot benefit from packages, but at least they profit from having solid access structures, which are of course important to PL/SQL too.

One important thing to keep in mind is that you should always strive to write efficient, legible code, but that premature optimization is not the way to go.
Premature optimization involves tinkering with access structures and execution plans; it does not include simplifying, refactoring and rewriting queries in ways that enable Oracle to optimally use the database objects involved.

Rewriting queries with or without hints and studying the corresponding execution plans is tedious and best left for `high-impact SQL`_ only: queries that process many rows, have a high number of buffer gets, require many disk reads, consume a lot of memory or CPU time, perform many sorts, and/or are executed frequently.
You can identify such queries from the dynamic performance views.
Whatever you, the database developer, do, be consistent and document your findings, so that all developers on your team may benefit from your experiences.

.. _`it is recommended`: http://www.dba-oracle.com/art_sql_tune.htm
.. _`high-impact SQL`: http://www.dba-oracle.com/art_sql_tune.htm