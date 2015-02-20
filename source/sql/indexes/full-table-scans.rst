.. _sql-indexes-fts:

Full Table Scans
================
Full table scans are often seen as a database's last resort: you only do them if you absolutely have to.
That reputation of full table scans is not entirely warranted though.
 
For small tables it often does not make sense for Oracle to read the associated index, search for the relevant ROWIDs, and then fetch the data from the database tables when it can just as easily do a single round trip to the table.
Thanks to multi-block I/O in full table scans a couple of parallel round trips are also possible to speed up the process.
 
Analogously, when the database has to return a sizeable portion of all the rows from a table, the index lookup is an overhead that does not always pay off.
It can even make the database jump back and forth between blocks.
 
Full table scans frequently indicate that there is optimization potential but remember, as originally noted by `Tom Kyte`_: "full table scans are not evil, indexes are not good".

.. _`Tom Kyte`: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:9422487749968