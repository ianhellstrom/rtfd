.. _plsql-cache-alt-det-vs-rc:

``DETERMINISTIC`` vs ``RESULT_CACHE``
-------------------------------------
A common question with caching is whether the ``DETERMINISTIC`` option or the ``RESULT_CACHE`` is best.
As always, the answer is: 'It depends.'
 
When you call a deterministic function many times from within the *same* SQL statement, the ``RESULT_CACHE`` does not add much to what the ``DETERMINISTIC`` option already covers.
Since a single SQL statement is executed from only one session, the function result cache cannot help with multi-session caching as there is nothing to share across sessions.
As we have said, marking a deterministic function as ``DETERMINISTIC`` is a good idea in any case.
 
When you call a deterministic function many times from *different* SQL statements — in potentially different sessions or even `instances of a RAC`_ — and even PL/SQL blocks, the ``RESULT_CACHE`` does have benefits.
Now, Oracle can access a single source of cached data across statements, subprograms, sessions, or even application cluster instances.
 
The 'single source of cached data' is of course only true for DR units.
For IR units, the function result cache is user-specific, which probably dampens your euphoria regarding the function result cache somewhat.
Nevertheless, both caching mechanisms are completely handled by Oracle Database.
All you have to do is add a simple ``DETERMINISTIC`` and/or ``RESULT_CACHE`` to a function's definition.
 
.. _`instances of a RAC`: http://www.oracle.com/technetwork/articles/datawarehouse/vallath-resultcache-rac-284280.html

