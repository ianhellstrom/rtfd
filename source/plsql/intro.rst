.. _plsql-intro:

***************
What is PL/SQL?
***************
PL/SQL stands for Procedural Language/Structured Query Language and it is Oracle's extension of SQL that aims to `seamlessly process SQL commands`_. Because it has been designed to integrate with SQL it supports the same data types. Stored PL/SQL source code is compiled and saved in the Oracle database. With PL/SQL you have access to common 3GL constructs such as conditional blocks, loops, and exceptions. IBM DB2 supports PL/SQL (as of version 9.7.0) and PostgreSQL has a procedural language called PL/pgSQL that borrows heavily from PL/SQL but is not compliant.

Each PL/SQL unit at schema level is a piece of code that is compiled and at some point executed on the database. Each unit can be one of the following: anonymous block, function, library, package, package body, procedure, trigger, type, or type body.

You can send a PL/SQL block of multiple statements to the database, thereby reducing traffic between the application and the database. Furthermore, all variables in ``WHERE`` and ``VALUES`` clauses are turned into bind variables by the PL/SQL compiler. Bind variables are great as they allow SQL statements to be reused, which can improve the performance of the database. We shall talk more about bind variables later (see :ref:`plsql-bind-vars`).

Both static and dynamic SQL are supported by PL/SQL.

.. _seamlessly process SQL commands: http://www.oracle.com/technetwork/database/features/plsql/index.html

As aptly phrased `here <http://stackoverflow.com/a/1252884>`_ on Stackoverflow: nothing is faster than work you don't do.
