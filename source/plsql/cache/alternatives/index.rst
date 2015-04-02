.. _plsql-cache-alt:
 
Alternatives
============
Oracle offers a few alternatives for caching: deterministic functions, package-level variables, and the result cache (for functions).
Each of these techniques has its pros and cons.
 
Package-level variables are conceptually the simplest but require the developer to think about and manage the cache.
A package-level variable is really what it says on the box: a variable in a package that contains static (reference) data that needs to be accessed many times.
 
Suppose you have an expensive deterministic function that you need to execute many times.
You can run the function for different parameter values and store the combination of input-output as a key-value pair in a collection at the package level.
You can thus access the data without having to recompute it every time.
 
Similarly, you can store (pre-aggregated) data from a table that does not change while you work with it, so you don't have to go back and forth between the disk and the PGA.
Alternatively, you can create a just-in-time cache, where each time a new value or row is requested, you check the cache and return the data (if already present) or add it to the cache (if not yet available in the cache) for future use.
That way, you do not create a huge collection up front but improve the lookup performance of subsequent data access requests or computations.
 
Another common use case for package-level variables that has little to do with caching though is when people try to avoid the dreaded `mutating table error with triggers`_.
 
A problem with package-level variables is that they are stored in the :term:`UGA`.
This means that the memory requirements go up as there are more sessions.
Of course, you can add the ``SERIALLY_REUSABLE`` directive to the package to reduce storage requirements, but it also means that the data needs to be exactly the same across sessions, which is rare, especially in-between transactions.
 
You do *not* want to use package-level variables as caches when the data contained in them changes frequently during a session or requires too much memory.
A measly 10 MB per user quickly becomes 1 GB when there are 100 sessions, making a what seemed to be a smart idea a veritable memory hog.
 
In terms of speed, packaged-based caching beats the competition, although the function result cache is `a close second`_.
But, as we have mentioned, package-based caching is also fairly limited in its uses and requires a developer to think carefully about the caching mechanism itself and the overall memory requirements.
Moreover, on Oracle Database 11g and above, the built-in alternatives are almost always a better option.
 
.. _`mutating table error with triggers`: http://oracle-base.com/articles/9i/mutating-table-exceptions.php#solution_1
.. _`a close second`: http://www.oracle-developer.net/display.php?id=504



.. toctree::
   :hidden:

   DETERMINISTIC Functions <deterministic>
   The RESULT_CACHE Option <result-cache>
   DETERMINISTIC vs RESULT_CACHE <deterministic-vs-result-cache>