.. _model-partition-caveats:

Caveats
=======
There are a couple of `impossible situations`_ with partitioned indexes.
First, you cannot have a local non-prefixed unique index.
Why?
Well, to check uniqueness Oracle would have to check every index partition, which may be prohibitively slow.
Unique indexes can only be global or local prefixed, unless the index contains `a subset of the partition key`_.

Second, we have already hinted at it: you cannot have a global non-prefixed index.
Why?
Well, you can simply create a concatenated global index on the columns that you want to index on and partition by.
Suppose you want to partition the fridge's index on product by expiry date; this would be useful for queries that look for a particular products and a range of expiry dates.
Queries with only the expiry date would have to search each partition of such a global index, which amounts to an index skip scan of the non-partitioned index, where the leading edge is the product.
Therefore, a global non-prefixed index would not give you anything that a concatenated index on product and expiry date does not already cover.
Oracle has thus done us all a favour by supplying fewer moving parts.

Third, there are no global partitioned bitmap indexes.
Concurrent DML causes entire blocks of a global bitmap index to be locked, which in turn means locks on potentially many, many rows.
Sure, bitmap indexes are typically recommended for data warehouses, which sort of goes against the idea of global partitioned indexes, which are the go-to indexes for partitioned tables in OLTP systems.
Global partitioned bitmap indexes would offer no advantages while only adding complexity and performance hits.

.. _`impossible situations`: http://www.orafaq.com/node/2833
.. _`a subset of the partition key`: http://logicalread.solarwinds.com/oracle-11g-partitioned-indexes-mc02

