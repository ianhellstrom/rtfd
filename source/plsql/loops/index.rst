.. _plsql-loops:

***********************************
Loops, Cursors, and Bulk Operations
***********************************
Oracle generously provides a list of things developers can do to `tune their PL/SQL code`_.
One item from that list is probably the single best tool developers have at their disposal to supercharge their code: bulk processing.
 
Whenever you need to retrieve and modify more than one row of a database table you have a few options:
 
* A single SQL statement.
* A cursor ``FOR`` loop to iterate through the results one row at a time to insert, update and/or delete records.
* Retrieve and temporarily store all records in a collection (``BULK COLLECT``), and process the rows
 
  * one at a time with a ``FOR`` loop;
  * in bulk with ``FORALL``.
 
* Retrieve all records in batches with an explicit cursor and store the results in a collection (``BULK COLLECT ... LIMIT``), after which the rows are processed
 
  * one at a time with a ``FOR`` loop;
  * in bulk with ``FORALL``.
 
Even though it is possible to use a single SQL statement to modify data in a table, PL/SQL developers rarely take this approach.
One of the primary reasons against such a single statement is that it is an all-or-nothing proposition: if anything goes wrong all modifications are rolled back.
With PL/SQL you have more control over what happens when exceptions occur.
 
The advantage of ``FORALL`` is that the statements are sent (in batches) from PL/SQL to the SQL engine `in one go`_, thereby minimizing the number of :term:`context switches <context switch>`.
A disadvantage is that with ``FORALL`` you can only throw one DML statement over the fence to the SQL engine, and the only differences allowed are in the ``VALUES`` and ``WHERE`` clauses of the modification statement.
Beware that when multiple ``INSERT`` statements are sent to the SQL statement executor with ``FORALL``, any statement-level triggers will fire only once: either before all statements have been executed or after all statements have completed.
 
``BULK COLLECT`` does for queries what ``FORALL`` does for :term:`IUD statements`; ``MERGE`` is supported from Oracle Database 11g onwards.
Since `collections`_ are required in ``BULK COLLECT`` and ``FORALL`` statements, and (pipelined) table functions, where they can greatly improve the runtime performance, we take a moment to go through the fundamentals.

.. _`tune their PL/SQL code`: http://docs.oracle.com/database/121/LNPLS/tuning.htm#LNPLS012
.. _`in one go`: http://www.orafaq.com/node/1399
.. _`collections`: http://www.oracle.com/technetwork/issue-archive/2012/12-jul/o42plsql-1653077.html


.. toctree::
   :hidden:

   Collections <collections>
   Performance Comparisons <performance>
   Caveats <caveats>