.. _model-compression:

***********
Compression
***********
Some compression options are not available to all database versions. 
`Oracle Advanced Compression`_ needs to be purchased as a separate option.
Moreover, a few compression methods are not even `available`_ to the enterprise edition, only to Oracle Exadata.

Although there is more to Oracle's Advanced Compression add-on, we shall focus on pure data compression and not ``SECUREFILE`` LOB deduplication or network compression although both are important in their own right.

Oracle promises customers a storage space reduction factor of `two to four`_ with advanced row compression. 
So, if your database is running low on hard drive space and you desperatly need to make some space for more data, you might benefit from advanced row compression. 
Sure, we recommend you take these numbers with a grain of salt as they are listed in company propaganda pamphlets, but still: compression will reduce the storage required to save the data. 
How much obviously depends on the contents of the data.

There is no such thing as a free lunch or compression without a catch. 
It's not as bad as you may think though: Oracle is able to *read* compressed blocks directly, that is, without having to decompress them. 
This means that there is *no* performance hit when reading data. 
Sweet!

So, the main problem lies with modifying data, and even there Oracle has stuffed a bunch of tricks up its sleeve. 
Blocks are in fact decompressed in batch mode, which means that a freshly initialized block remains uncompressed until an internally controlled threshold is reached. 
Once the threshold is reached, Oracle compresses the block. 
When more data is dumped into the block, the same game is played again: once the threshold is reached, the *entire* block is recompressed to achieve the greatest savings in storage.

So, when would you consider using advanced row compression?

In data warehouses or OLAP environments compression makes a lot of sense. 
Most operations are read-only and they touch upon lots of data. 
Since reading does not cause additional I/O, it seems almost silly not to compress the data. 
In fact, table scans can be performed more efficiently because compressed blocks Oracle are smaller than uncompressed blocks.

In OLTP databases the situation is quite similar although some care needs to be taken. 
Remember that blocks are compressed and decompressed *in batches*, so that no single row-level transaction has a performance hit. 
The CPU usage will spike `when the compression algorithm is triggered`_ but that typically happens after many single-row transactions.

Advanced row compression is not the only kid on the block. 
Hybrid columnar compression (HCC) is available in Oracle Exadata, and it uses a combination of column- and row-level compression because values may be repeated many times *across* rows and *within* columns rather than rows. 

Before we go into specifics and discuss performance considerations, let's take a step back and see how Oracle's compression algorithms work.

.. _`Oracle Advanced Compression`: http://www.oracle.com/us/products/database/options/advanced-compression/overview/index.html
.. _`available`: http://docs.oracle.com/database/121/DBLIC/editions.htm#DBLIC116
.. _`two to four`: http://www.oracle.com/technetwork/database/options/compression/advanced-compression-wp-12c-1896128.pdf
.. _`when the compression algorithm is triggered`: http://www.oracle.com/au/products/database/11g-compression-198295.html



.. toctree::
   :hidden:

   Compression Methods <methods>
   Performance Considerations <performance>