.. _model-partition:

************
Partitioning
************
Partitioning is often the key to improved application performance for very large databases (VLDBs).
The reason is simple: the data is split into chunks and each partition behaves a little like a table itself: it has its own name and storage characteristics.
Each partition is managed and operated independently from all other partitions of the same table.
Physically, the data for different partitions is stored in separate segments; logically, the various partitions still make up the entire table and SQL statements still look the same as before.
Moreover, no segments are allocated for the table itself, which is nothing but a collection of partitions.

The main advantages of partitioning that `Oracle lists`_ are:

* increased availability
* easier administration of schema objects
* reduced contention for shared resources in OLTP systems
* enhanced query performance in data warehouses

A common scenario in a data warehouse is to run queries against a table for a particular period of (historical) time.
Creating a (range) partition on the time column of such a table may thus enable you to query the table in a more efficient way, as Oracle only has to access one particular partition rather than the whole table.

There are three partitioning strategies:

#. Range partitioning: rows are partitioned based on ranges of the partition key, which can be one or more columns of a table. 
   A prime example of such a strategy is for time ranges.
#. List partitioning: rows are partitioned based on a (discrete) list of partition key values.
   A use case of list partitioning is for identifier values, for instance countries or regions.
#. Hash partitioning: rows are partitioned with a hashing algorithm; the hash function determines in which partition a particular row ends up.
   High update contention in OLTP applications can be resolved with hash partitioning as the :term:`IUD <IUD statements>` operations can be for different segments; I/O bottlenecks can be reduced or avoided because the data is `randomly distributed across partitions`_.

The first two partitioning strategies require you to define names for each partition and perhaps use the ``ALTER TABLE ... ADD PARTITION ...`` statement to add more partitions as time progresses.
The third partitioning strategy requires you to define the partition key and the number of partitions; Oracle takes care of the rest.
Partition names for hash-partitioned tables are automatically generated.

A table can be either partitioned by one of the three partitioning strategies or by a combination of these.
In the former case we speak about single-level partitioning.
Composite partitioning is what the latter case is called because each partition is subdivided into a subpartition.
For example, a table can be partitioned on the ``SALES_DATE`` and then subpartitioned on ``SALES_REGION``.
When you typically only run queries for, say, a time range of a month and a particular sales region, these queries may return the result set a lot faster as the data is contained in one subpartition.

A caveat of partitioning is that a non-partitioned table cannot be made into a partitioned table with a simple ``ALTER TABLE`` statement.
You have to recreate the entire table.

Heap-organized (i.e. 'normal') tables can be stored in compressed format, which may be beneficial to query execution speed, not to mention storage requirements.
Partition compression is typically only useful in OLAP systems, such as data warehouses, where IUD operations are much rarer than queries.

Note that `virtual columns`_ can be used to partition a table on.
What is not allowed, though, are PL/SQL function calls in the definition of the virtual column.

.. _`Oracle lists`: http://docs.oracle.com/database/121/CNCPT/schemaob.htm
.. _`randomly distributed across partitions`: http://docs.oracle.com/database/121/VLDBG/GUID-F023D3ED-262F-4B19-950A-D3C8F8CDB4F4.htm
.. _`virtual columns`: http://docs.oracle.com/database/121/VLDBG/GUID-811EDE81-7016-43AF-9078-A435DD52EFA5.htm



.. toctree::
   :hidden:

   Partitioned Indexes <partitioned-indexes>
   Caveats <caveats>
   Recommendations <recommendations>

