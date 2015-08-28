.. _model-compression-index:

Compression Methods
===================
Let's take a look at the different compression methods Oracle offers.


``BASIC``  and ``OLTP`` 
-----------------------
Important to note is that for ``BASIC`` and ``OLTP`` Oracle deduplicates data in rows in one block rather than compression in the classical sense. 
Oracle can internally `rearrange the order of the columns`_ to combine deduplication symbols and save more disk space.

Because of this Oracle does not 'decompress' the blocks, it simply reconstructs the original rows from reading the row bits (with the symbols) and combining that information with the symbol directory that contains the token or symbol values. 
This back-and-forth may cause significant CPU time, although it may be offset against a reduction in the number of physical reads because the same data is available in a smaller number of blocks.

When data is deleted from a table with ``BASIC`` compression, Oracle must `maintain the symbol table`_. 
Once a symbol has no references any longer, the symbol itself is removed.

Tables defined with ``BASIC`` compression automatically have ``PCTFREE`` set to zero.
In fact, whenever you ``MOVE`` the table, Oracle resets the value to zero.
So, when you update column values and Oracle expands the symbols, there typically is not much space.
This in turn means that when you run many updates on the same block, Oracle may have to migrate rows, which causes a performance degradation.
An appropriate level of ``PCTFREE``, which has to be reset every time you move tables, may alleviate the situation.

Another problem with ``UPDATE`` statements on tables with basic compression is that Oracle works on a copy of the row that has the column values modified and fully expanded, that is, decompressed.
To add insult to injury, Oracle does not recompress these modified rows, irrespective of the presence of a suitable symbol in the directory.

Although ``OLTP`` ought to solve the ``UPDATE`` problem of the basic compression, `it does not`_.

Important to note is that a standard ``INSERT`` does *not* compress data even when the target table has compression enabled. 
Only CTAS statements, direct-path or parallel inserts, and ``ALTER TABLE tab_name COMPRESS ...`` with a subsequent ``ALTER TABLE tab_name MOVE`` cause fresh data to become stored in compressed format.

Hybrid Columnar Compression
---------------------------
Advanced row compression looks in each row and creates smaller symbols that represent repeated values in the same row. 
Sure, this may be useful but let's not kid ourselves: in real databases the number of rows where, say, the first name of an employee matches the city he or she lives in is fairly limited.

That observation is the kick-off for hybrid columnar compression. 
Since typically values are repeated in the same column, it makes more sense to create symbols for columns rather than rows. 
That is exactly what HCC does. Instead of having a symbol directory for each block, Oracle can even get away with a single symbol directory for each column.

``QUERY { LOW | HIGH }``
^^^^^^^^^^^^^^^^^^^^^^^^
These two HCC options are ideal for data warehouses. 
The 'low' setting is best suited for databases where the ETL is a critical factor, whereas 'high' is mainly reserved for situations where the space savings are paramount.

``ARCHIVE { LOW | HIGH }``
^^^^^^^^^^^^^^^^^^^^^^^^^^
Whenever data needs to be archived but retrieval is not really a main concern, for instance for historical data that is needed for compliance but otherwise not really interesting to the business, the archival compression options are a good choice. 
The 'low' and 'high' options follow the same logic as before: if loading the data is a performance concern, then lowering the compression rate may be give you an edge in the ETL processes, otherwise go for glory with the highest available compression to save most on disk space.

.. _`rearrange the order of the columns`: http://allthingsoracle.com/compression-oracle-basic-table-compression
.. _`maintain the symbol table`: http://allthingsoracle.com/compression-in-oracle-part-2-read-only-data
.. _`it does not`: http://allthingsoracle.com/compression-in-oracle-part-3-oltp-compression