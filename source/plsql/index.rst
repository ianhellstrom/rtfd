.. _plsql-intro:

######
PL/SQL
######

PL/SQL stands for Procedural Language/Structured Query Language and it is Oracle's extension of SQL that aims to `seamlessly process SQL commands`_. 
One reason why PL/SQL is so important to database and application developers is that SQL itself offers no robust procedural constructs to apply logical processing to DML statements.
Because it has been designed to integrate with SQL it supports the same data types. 
Stored PL/SQL source code is compiled and saved in the Oracle database. 
With PL/SQL you have access to common 3GL constructs such as conditional blocks, loops, and exceptions. 
IBM DB2 supports PL/SQL (as of version 9.7.0) and PostgreSQL has a procedural language called PL/pgSQL that borrows heavily from PL/SQL but is not compliant.

Each PL/SQL unit at schema level is a piece of code that is compiled and at some point executed on the database. 
Each unit can be one of the following: anonymous block, function, library, package (specification), package body, procedure, trigger, type, or type body.

You can send a PL/SQL block of multiple statements to the database, thereby reducing traffic between the application and the database. 
Furthermore, all variables in ``WHERE`` and ``VALUES`` clauses are turned into bind variables by the PL/SQL compiler. 
Bind variables are great as they allow SQL statements to be reused, which can improve the performance of the database. 
We shall talk more about bind variables later (see :ref:`plsql-bind`).

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

SQL statements are often all-or-nothing propositions: if only one row does not satisfy, say, a constraint, the entire data set may be rejected.
With PL/SQL you can insert, update, delete, or merge data one row at a time, in batches whose size you can determine, or even all data at once just like in SQL.
Because of this fine-grained control and the PL/SQL's exception handling capabilities you can develop applications that provide meaningful information to users in case something goes wrong rather than a cryptic ``ORA-00001: unique constraint ... violated`` or a equally uninformative ``ORA-02290 error: check constraint ... violated``.
Similarly, PL/SQL allows database developers to create applications that do exactly the work that is needed and nothing more.
After all, as aptly phrased `here <http://stackoverflow.com/a/1252884>`_ on Stackoverflow: nothing is faster than work you don't do.

Anyway, time to sink our teeth into PL/SQL...

.. _`seamlessly process SQL commands`: http://www.oracle.com/technetwork/database/features/plsql/index.html
.. _`most notably`: http://www.amazon.com/Study-Guide-1Z0-144-Database-Certification/dp/1478217995

Compilation
===========
When a user sends a PL/SQL block from a client to the database server, the PL/SQL compiler translates the code, assuming it is valid, into bytecode.
Embedded SQL statements are modified before they are sent to the SQL compiler.
For example, ``INTO`` clauses are removed from ``SELECT`` statements, local variables are replaced with bind variables, and many identifiers are written in upper case.
If the embedded SQL statements contain any PL/SQL function calls, the SQL engine lets the PL/SQL engine take over to complete the calls in case the function result cache does not have the desired data.
Once the SQL compiler is done, it hands the execution plan over to the SQL engine to fulfil the request.
Meanwhile the :term:`PVM` interprets the PL/SQL bytecode and, once the SQL engine has come up with the results, returns the status (success/failure) to the client session.

Whenever an identifier cannot be resolved by the SQL engine, it escapes, and the PL/SQL engine tries its best to resolve the situation.
It first checks the current PL/SQL unit, then the schema, and if that does not yield success, it eventually generates a compilation error.
 
The `PL/SQL compilation` itself proceeds as follows:
 
#. Check syntax.
#. Check semantics and security, and resolve names.
#. Generate (and optimize) bytecode (a.k.a. MCode).
 
PL/SQL is derived from Ada, and uses a variant of DIANA: Distributed Intermediate Annotated Notation for Ada, a tree-structured intermediate language.
DIANA is used internally after the syntactic and semantic check of the PL/SQL compilation process.
The DIANA code is fed into the bytecode generator for the PL/SQL Virtual Machine (PVM) or compiled into native machine code (C), depending on the setting of ``PLSQL_CODE_TYPE``.
For native compilation `a third-party C compiler`_ is required to create the DLLs (Windows) or shared object libraries (Linux).
In the final step, the PL/SQL optimizer may, depending on the value of ``PLSQL_OPTIMIZE_LEVEL``, whip your code into shape.
What the optimizer can do to your code is described `in a white paper`_.
 
When the MCode or native C code is ready to be executed, Oracle loads it into either the shared pool (MCode) or the PGA (native).
Oracle then checks the ``EXECUTE`` privileges and resolves external references.
MCode is interpreted and executed by the PL/SQL runtime engine; native code is simply executed.
 
Both the DIANA and bytecode are stored in the data dictionary.
For anonymous blocks the DIANA code is discarded.
 
.. _`PL/SQL compilation`: http://eu.wiley.com/WileyCDA/WileyTitle/productCd-0764598074.html
.. _`a third-party C compiler`: http://www.cengage.com/search/productOverview.do?N=16+4294922389+4294966055+19&Ntk=P_EPI&Ntt=11395859158516404884848709101527959663
.. _`in a white paper`: http://www.oracle.com/technetwork/database/features/plsql/codeorder-133512.zip



.. toctree::
   :hidden:

   bind/index
   loops/index
   cache/index