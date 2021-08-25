.. _plsql-cache-nocopy:
 
The ``NOCOPY`` Directive: To Pass By Value or Reference?
========================================================
In many programming languages you can pass parameters by value or reference, which can be as different as night and day in terms of performance.
The reason is simple: a large object that is passed by value needs to be copied, which may take a considerable amount of time, whereas a reference (i.e. pointer) to that object is about as cheap as chips.
 
 
When you specify formal parameters to subprograms you cannot only define ``IN``, ``OUT``, and ``IN OUT``, but also ``NOCOPY``.
What this does is *ask* the compiler to pass the actual parameters `by reference rather than value`_.
Please note the emphasis on 'ask': it is not a directive, it is merely a request.
 
``IN`` parameters are *always* passed by reference.
When a parameter is passed by reference, the formal and actual parameters refer to the same memory location.
Any changes to the value of the formal parameter are immediately reflected in the actual parameter as well.
Aliasing is not an issue for ``IN`` parameters, because subprograms cannot assign values to them.
 
``OUT`` and ``IN OUT`` parameters can be passed by reference or value.
Hence, aliasing can become an issue, which means that assignments `may or may not`_ show up immediately in all parameters, especially when the same parameter is `passed by reference as well as value`_ in different places: when a pass-by-value parameter is copied back to the original data structure that is also passed by reference elsewhere, the modifications will overwrite any changes already done by the passed-by-reference subprograms.
Moreover, if the subprogram raises an exception, there is no way to undo or rollback the changes made to the data structure, because there is no copy of it from before all the modifications.
 
The ``NOCOPY`` hint can thus help you reduce unnecessary CPU cycles and memory consumption: by adding ``NOCOPY`` you request that the compiler only pass the memory address rather than the entire data structure.
 
There are several cases where you can be sure the compiler ignores ``NOCOPY``:
 
* The actual parameter must be *implicitly* converted to the data type of the formal parameter.
* The actual parameter is the element of a collection (unless it is also a collection -- see dbms_sql.column_value procedure).
* The actual parameter is a scalar variable with a ``NOT NULL`` constraint.
* The actual parameter is a scalar numeric variable with a range, size, or precision constraint.
* The actual *and* formal parameters are records.
* The actual and/or formal parameters are declared with ``%ROWTYPE`` or ``%TYPE``, and the constraints differ.
* The code is called through a database link or external program.
 
In summary, you may see a `performance boost`_ from using ``NOCOPY`` but be careful: when you run into issues, the original data structure may be corrupted!
 
.. _`by reference rather than value`: http://docs.oracle.com/database/121/LNPLS/formal_parameter.htm#CJACJCGJ
.. _`may or may not`: http://docs.oracle.com/database/121/LNPLS/subprograms.htm#LNPLS00815
.. _`passed by reference as well as value`: http://oracle-base.com/articles/misc/nocopy-hint-to-improve-performance-of-parameters-in-plsql.php
.. _`performance boost`: http://www.dba-oracle.com/plsql/t_plsql_nocopy_hint.htm
