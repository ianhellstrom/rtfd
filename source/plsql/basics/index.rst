.. _plsql-basics-compilation:

***********
Compilation
***********
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