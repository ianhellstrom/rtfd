.. _sql-indexes-iot:
 
Index-Organized Tables
======================
Index-organized tables are generally narrow lookup tables.
They have to be narrow because all columns of the table are in the index.
In fact, the index is the table itself.
 
It is technically possible to add additional indexes to an index-organized table.
However, accessing an index-organized table via a secondary index is very inefficient.
The reason is that the secondary index cannot have pointers to rows in the table because that would require the data to stay where it is.
Forever.
Because the data is organized by the primary index in an index-organized table, it can move around whenever data in modified.
Secondary indexes store logical instead of physical ROWIDs; `logical ROWIDs`_ (``UROWID``) contain physical guesses, which identify the block of the row at the time when the secondary index was created or rebuilt.
Standard heap tables are generally best for tables that require multiple indexes.
 
Index-organized tables can be beneficial to OLTP applications where fast primary-key access is essential; inserts typically take longer for index-organized tables.
Because the table is sorted by the primary key, duplication of data structures is avoided, reducing storage requirements.
Key compression, which breaks an index key into a prefix and suffix, of which the former can be shared among the suffix entries within an index block, reduces disk storage requirements even further.
 
Index-organized tables cannot contain virtual columns.

.. _`logical ROWIDs`: http://docs.oracle.com/database/121/CNCPT/indexiot.htm#CNCPT911