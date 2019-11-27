.. _plsql-cache-udf:
 
The ``UDF`` Pragma
==================
The ``PRAGMA UDF`` `compiler directive`_ is not a caching mechanism.
'So, what's it doing in the chapter on caching?' we hear you ask.
 
It's an optimization technique for subprograms, so it fits in nicely into our current discussion.
It tells the compiler that the function ought to be prepared for execution from SQL statements.
Because of that information, Oracle can sometimes reduce the cost of context switches.
 
As of Oracle Database 12c, there is also the possibility of adding a PL/SQL function to your SQL statement with the ``WITH`` clause.
A non-trivial example is described on `Databaseline`_, from which it follows that the ``WITH`` clause is `marginally faster`_ than the ``UDF`` pragma, but that the latter has the advantage that it is modular, whereas the former is the equivalent of hard coding your functions.
 
We can therefore recommend that you first try to add ``PRAGMA UDF`` to your PL/SQL functions *if and only if* they are called from SQL statements but not PL/SQL code.
If that does not provide a significant benefit, then try the ``WITH`` function block.
 
.. _`compiler directive`: http://docs.oracle.com/database/121/LNPLS/udf_pragma.htm
.. _`Databaseline`: http://www.databaseline.tech/how-to-multiply-across-a-hierarchy-in-oracle-part-1/
.. _`marginally faster`: http://www.databaseline.tech/how-to-multiply-across-a-hierarchy-in-oracle-part-2/