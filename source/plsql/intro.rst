.. _plsql-intro:

***************
What is PL/SQL?
***************
PL/SQL stands for Procedural Language/Structured Query Language and it is Oracle's extension of SQL that aims to `seamlessly process SQL commands`_. 
One reason why PL/SQL is so important to database and application developers is that SQL itself offers no robust procedural constructs to apply logical processing to DML statements.
Because it has been designed to integrate with SQL it supports the same data types. 
Stored PL/SQL source code is compiled and :term:`saved <PVM>` in the Oracle database. 
With PL/SQL you have access to common 3GL constructs such as conditional blocks, loops, and exceptions. 
IBM DB2 supports PL/SQL (as of version 9.7.0) and PostgreSQL has a procedural language called PL/pgSQL that borrows heavily from PL/SQL but is not compliant.

Each PL/SQL unit at schema level is a piece of code that is compiled and at some point executed on the database. 
Each unit can be one of the following: anonymous block, function, library, package (specification), package body, procedure, trigger, type, or type body.

You can send a PL/SQL block of multiple statements to the database, thereby reducing traffic between the application and the database. 
Furthermore, all variables in ``WHERE`` and ``VALUES`` clauses are turned into bind variables by the PL/SQL compiler. 
Bind variables are great as they allow SQL statements to be reused, which can improve the performance of the database. 
We shall talk more about bind variables later (see :ref:`plsql-bind-vars`).

Both static and dynamic SQL are supported by PL/SQL.
This allows developers to be as flexible as one needs to be: statements can be built on the fly.
The availability of collections, bind variables, and cached program modules is key to applications with scalable performance.
PL/SQL even supports object-oriented programming by means of abstract data types.

Each unit must at least contain a ``BEGIN`` and matching ``END``, and one executable statement, which may be the ``NULL;`` statement.
The declaration and exception handling sections are optional.
For stored PL/SQL units, such as functions, procedures, and packages, you also need a header that uniquely identifies the unit by means of a name and its signature.

Although PL/SQL and SQL are close friends, not all SQL functions are available in PL/SQL, `most notably`_:

* aggregate and analytic functions, e.g. ``SUM`` and ``MAX``
* model functions, e.g. ``ITERATION_NUMBER`` and ``PREVIOUS``
* data mining functions, e.g. ``CLUSTER_ID`` and ``FEATURE_VALUE``
* encoding and decoding functions, e.g. ``DECODE`` and ``DUMP``
* object reference functions, e.g. ``REF`` and ``VALUE``
* XML functions, e.g. ``APPENDCHILDXML`` and ``EXISTSNODE``
* ``CUBE_TABLE``
* ``DATAOBJ_TO_PARTITION``
* ``LNNVL``
* ``NVL2``
* ``SYS_CONNECT_BY_PATH``
* ``SYS_TYPEID``
* ``WIDTH_BUCKET``

Most of these functions involve multiple rows, which is a key indicator that it cannot work in pure PL/SQL, which operates on single rows.
Of course, you *can* use these functions in SQL statements embedded in PL/SQL units.

This does not mean that PL/SQL can handle only one row at a time: bulk operations, such as ``BULK COLLECT`` and ``FORALL``, are necessary to achieve solid performance when more than one row is involved, as rows are processed in batches.
These bulk operations cause fewer context switches between the SQL and PL/SQL engine.
Context switches add a slight overhead that can become significant the more data needs to be processed.

SQL statements are often an all-or-nothing proposition: if only one row does not satisfy, say, a constraint, the entire data set may be rejected.
With PL/SQL you can insert, update, delete, or merge data one row at a time, in batches whose size you can determine, or even all data at once just like in SQL.
Because of this fine-grained control and the PL/SQL's exception handling capabilities you can develop applications that provide meaningful information to users in case something goes wrong rather than a cryptic ``ORA-00001: unique constraint ... violated`` or a equally uninformative ``ORA-02290 error: check constraint ... violated``.
Similarly, PL/SQL allows database developers to create applications that do exactly the work that is needed and nothing more.
After all, as aptly phrased `here <http://stackoverflow.com/a/1252884>`_ on Stackoverflow: nothing is faster than work you don't do.

Anyway, time to sink our teeth into PL/SQL...

.. _seamlessly process SQL commands: http://www.oracle.com/technetwork/database/features/plsql/index.html
.. _most notably: http://www.amazon.com/Study-Guide-1Z0-144-Database-Certification/dp/1478217995
