.. _plsql-loops-caveats:

Caveats
=======
We have already mentioned that ``FORALL`` can only be used when one DML statement needs to be executed.
Unfortunately, that's not where the bad news ends.
 
Any *unhandled* exception causes *all* IUD changes to be rolled back.
The exception to this is when exceptions are managed with the ``SAVE EXCEPTIONS`` clause.
If there are failures that are saved, Oracle will raise a single exception (``ORA-24381``) for the entire statement after completing the ``FORALL``.
You can query the pseudo-collection ``SQL%BULK_EXCEPTIONS`` to get the information about these exceptions.
 
.. note::
   Any collection index referenced in the IUD statement of a ``FORALL`` cannot be an expression: it must be the plain index itself.
   The collection index is implicitly defined by Oracle as a ``PLS_INTEGER``.
 
Another gotcha is that `bulk SQL`_ can only be performed on local tables: you cannot do a ``BULK COLLECT`` over a database connection.
Furthermore, parallel DML is disabled for bulk SQL.
 
As of Oracle Database 10gR2, the PL/SQL compiler `automatically optimizes`_ most cursor ``FOR`` loops into constructs that run with performance comparable to a ``BULK COLLECT``.
Yay!
Unfortunately, this does *not* happen automatically when there are IUD statements in your ``FORALL`` statements: these statements require manual intervention.
Boo!

For *pipelined* table functions we can define the collection types they return inside a package specification.
Oracle automatically creates database (i.e. schema-level) types based on the record and collection types defined in the package specification.
The reason Oracle does this is that table functions are invoked by the SQL engine, which does not know about PL/SQL constructs inside the package.
As such, Oracle must ensure that the SQL engine can do its work, so it creates the `shadow types`_ implicitly.
These types typically have names that start with ``SYS_PLSQL_``.
So, while you have all your types in the same file (i.e. package specification), which makes maintaining your code base easier, you end up with system-generated identifiers for types, which in itself can be a maintenance issue.

.. _`automatically optimizes`: http://www.toadworld.com/platforms/oracle/b/weblog/archive/2011/07/13/quicktips-using-bulk-collect-with-queries.aspx
.. _`shadow types`: http://oracle-base.com/articles/misc/pipelined-table-functions.php#implicit_types
.. _`bulk SQL`: http://www.oracle.com/technetwork/issue-archive/2012/12-sep/o52plsql-1709862.html