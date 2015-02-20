.. _sql-indexes-bitmap-indexes:
 
Beyond B-Trees: Bitmap Indexes
==============================
For columns with low cardinality the classical B-tree index is not an optimal solution, at least not in DSS or OLAP environments.
Bitmap indexes to the rescue!
 
Bitmap indexes use compression techniques, which means that many ROWIDs can be generated with very little I/O.
As argued by `Vivek Sharma`_ and `Richard Foote`_, a bitmap index is not only your go-to index for low-cardinality columns but also for any data that does not change frequently, for instance fact tables in data warehouses.
Nevertheless, concurrent IUD operations clearly tip the scales in favour of standard B-tree indexes; bitmap indexes are problematic for online applications with many concurrent DML statements because of deadlocks and the overhead to maintain bitmap indexes.
 
Ad hoc queries are also generally handled better by bitmap than B-tree indexes.
Queries with ``AND`` and ``OR`` can be executed efficiently because bitmap indexes on non-selective columns can be combined easily; ``COUNT`` queries are handled particularly efficiently by bitmap indexes.
If users query many different combinations of columns on a particular table, a B-tree index has no real candidate for the leading index column.
A bitmap index on all columns typically queried by analysts allows the index to be used for all these queries.
It does not matter whether your business users use only one, two, three, or all columns from the index in their queries.
In addition, nulls are included in the bitmap index, so you don't have to resort to function-based indexes.
 
By the way, you *can* create `bitmap indexes on index-organized tables`_.
More information on default B-tree and other indexes is of course `provided by Oracle`_.

.. _`Vivek Sharma`: http://www.oracle.com/technetwork/articles/sharma-indexes-093638.html
.. _`Richard Foote`: http://richardfoote.wordpress.com/2010/02/18/myth-bitmap-indexes-with-high-distinct-columns-blow-out
.. _`bitmap indexes on index-organized tables`: http://docs.oracle.com/database/121/ADMIN/tables.htm#ADMIN11699
.. _`provided by Oracle`: http://docs.oracle.com/database/121/ADMIN/indexes.htm#ADMIN11709