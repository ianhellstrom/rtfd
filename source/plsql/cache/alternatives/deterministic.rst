.. _plsql-cache-alt-det:

``DETERMINISTIC`` Functions
---------------------------
The ``DETERMINISTIC`` clause for functions is ideal for functions that do not have any non-deterministic components.
This means that each time you provide the function with the same parameter *values*, the result is the same.
When you define a function you can simply add the ``DETERMINISTIC`` option to the declaration section, making sure that the function (or any functions or procedures it calls) does not depend on the state of session variables or schema objects as the `results may vary across invocations`_.
This option instructs the optimizer that it may use a cached result whenever it encounters a previously calculated result.
 
Any speed-up from specifying a function as ``DETERMINISTIC`` becomes apparent when you execute the same function many times with the same parameter values *in the same SQL statement*.
A typical example is a user-defined conversion function that is called for each row of the result set of a query
 
Function-based indexes can only use functions marked ``DETERMINISTIC``.
The same goes for materialized views with ``REFRESH FAST`` or ``ENABLE QUERY REWRITE``.
 
One restriction is that you cannot define a nested function as deterministic.
 
Whenever a function is free of side effects and deterministic, it is a good practice to add the ``DETERMINISTIC`` option.
Whether the optimizer makes use of that information is entirely up to Oracle, but fellow coders will know that you intended to create a deterministic, side-effect free function, even when Oracle does not see a use for that information.
 
'What happens when I label a function as ``DETERMINISTIC`` but it secretly isn't?'
 
First off, why on earth would you do that?!
Second, Oracle cannot fix stupidity.
Its capability to discover non-deterministic ``DETERMINISTIC`` functions is not just limited, it is non-existent.
Yes, that's right: Oracle does not check whether you are telling the truth.
It trusts you implicitly.
 
What can happen, though, is that a user sees dirty (i.e. uncommitted) data because the incorrectly-marked-as-deterministic function queries data from a table that has been cached inappropriately.
`Neil Chandler`_ has written about the multi-version concurrency control model (MVCC) used by Oracle Database, which explains the roles of isolation levels, REDO, and UNDO with regard to `dirty reads`_, in case you are interested.
If you tell Oracle that a function is deterministic even though it is not, then the results can be unpredictable.
 
.. _`results may vary across invocations`: http://docs.oracle.com/database/121/LNPLS/function.htm
.. _`Neil Chandler`: http://chandlerdba.wordpress.com/2013/12/01/oracles-locking-model-multi-version-concurrency-control
.. _`dirty reads`: http://docs.oracle.com/database/121/CNCPT/consist.htm