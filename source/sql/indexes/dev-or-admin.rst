.. _sql-indexes-dev-or-admin:

Developer or Admin?
===================
Indexes can speed up lookups but having too many indexes causes serious performance degradations when inserting, updating, and deleting.
The reason is simple: the database has to maintain the index and the data structures associated with it.
As the index grows, branch and leaf nodes may have to be split, which obviously gobbles up valuable CPU time.
 
Horrible advice you'll sometimes encounter in your life as a database developer is that a DBA is responsible for indexes.
Absolutely not!
The performance of a ``SELECT`` depends on indexes, and the existence of indexes on a table affects ``INSERT``, ``UPDATE``,  and ``DELETE`` statements.
Only a developer knows what queries are typically run, how often a table's data is modified, how it is modified (i.e. single rows or in bulk, normal or direct-path inserts, …) so only a developer can judge whether an index on a particular combination of columns makes sense.
 
Knowledge of indexes is a must for every database developer.
A magnificent reference is Markus Winand's `SQL Performance Explained`_.
If it's not in your library, you're not yet serious about databases!
For the more frugal among us, his `website on indexes`_ is also a valuable resource that covers a lot of what's in the book.

.. _`SQL Performance Explained`: http://sql-performance-explained.com
.. _`website on indexes`: http://use-the-index-luke.com