.. _sql-hints-global:
 
Global Hints
============
Hints are commonly embedded in the statement that references the objects listed in the hints.
For hints on tables that appear inside views Oracle recommends using `global hints`_.
These hints are `not embedded in the view itself`_ but rather in the queries that run off the view, which means that the view is free of any hints that pertain to retrieving data from the view itself.
 
We shall presume that we have created a view called ``view_name``.
The view does a lot of interesting things but what we need for a global hint in our query that selects data from our view is a table ``tab_name`` inside a subquery (e.g. inline view or factored subquery) with the alias ``subquery_alias``.
We would then write ``SELECT /*+ SOME_HINT( view_name.subquery_alias.tab_name ) */ * FROM view_name``, where ``SOME_HINT`` is supposed to be any valid optimizer hint.
 
Similarly we could use a named query block to do the same: ``/*+ SOME_HINT( @my_qb_name  tab_name )``, where ``my_qb_name`` is the name we have given to the query block in which ``tab_name`` appears.
You can also use the automatically generated query block names but that is begging for trouble.
Named query blocks are really useful in conjunction with global hints.

.. _`global hints`: http://www.dba-oracle.com/t_sql_hints_tuning.htm
.. _`not embedded in the view itself`: http://docs.oracle.com/cd/B19306_01/server.102/b14211/hintsref.htm#i27644