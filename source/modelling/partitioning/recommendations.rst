.. _model-partition-recommendations:

Recommendations
===============
The following are rough guidelines that are intended to aid the developer in identifying the best partitioning strategy.

Single-Level Partitioning
-------------------------
Historical tables that are frequently scanned by range predicates and/or have a rolling window of data are ideal for range/interval partitioning.

Hash partitions are best for tables with heavy I/O, where the partition key is (almost) unique, so that the load can be distributed over several partitions.
The number of partitions is typically a power of two.

List partitioning is best used when there is a *single column* that is always searched for in ``WHERE`` filters and when the data must be mapped to different partitions based on a set of discrete values of that column.

Composite Partitioning
----------------------
Composite **range-hash** partitioning is common for huge tables that have historical data that are frequently joined with other large tables.
The main partition of range-hash partitioning enables historical analyses, such as long-running queries based on range predicates that can benefit from partition pruning, whereas the hash subpartitioning strategy allows :ref:`full or partial partition-wise joins <sql-join-partitions>` to be performed.
Similarly, tables that use hash partitioning but also need to maintain a rolling window are prime candidates for range-hash partitioning.

Composite **range-list** partitioning is best for historical data that is accessed by multiple dimensions.
One common access path is by, say, date, whereas another one is by country or region, or another discrete attribute.
The reverse, composite **list-range** partitioning, is the go-to partitioning scheme when the primary dimension that is used to access data is not a range or interval but a list.

Another option is composite **range-range** partitioning, which is ideal for tables with multiple time dimensions.
An example of multiple time dimensions is common in retail where one dimension lists when orders were placed and another one when they were shipped, invoiced, or paid.
Similarly, composite **list-list** partitioning is recommended for tables that are accessed by multiple discrete dimensions.

Yet another compositing partitioning strategy is **list-hash** partitioning.
The business case for this strategy is when typically the data is accessed through a the discrete (list) dimension but hashing is required to take advantage of parallel partition-wise joins that are done against another dimension that is (sub-)partitioned in the table(s) to be joined to the original one.

Prefixed vs Non-Prefixed Local Indexes
--------------------------------------
Oracle `recommends`_ that either prefixed local or global partitioned indexes be used for OLTP applications, as they minimize the number of `index partition probes`_.
Non-prefixed local indexes should be avoided in OLTP environments.
DSS or OLAP systems benefit mostly from local partitioned indexes, whereas global partitioned indexes are not really a great idea in these scenarios.
Non-prefixed local indexes are ideal for historical databases, especially when individual partitions need to be dropped regularly.

Partitioned vs Non-Partitioned Global Indexes
---------------------------------------------
When all the rows of a particular partition are (almost) always asked for, an index won't us do any good; the partition key is enough for Oracle to take advantage of `partition pruning`_.
In fact, the overhead of maintaining the index may cause unnecessary slowdowns when doing :term:`IUD <IUD statements>` operations.
When the partition key is absent from queries but we do filter for local index columns, Oracle needs to scan all partitions and use the local indexes to return the requested result set.
In such cases a global index is to be preferred.
Whether a partitioned or non-partitioned global index is best depends on the queries you run. 
If the index can be partitioned such that most queries only need to access a single partition, then partitioning is likely your best choice.
If not, you should default to a non-partitioned global index.

.. _`recommends`: http://docs.oracle.com/database/121/VLDBG/GUID-A43726D5-300D-4F5E-8FF3-85F057BC4CD3.htm
.. _`index partition probes`: http://docs.oracle.com/database/121/VLDBG/GUID-359C0752-D50C-4A93-9D6E-C1B67E81625D.htm
.. _`partition pruning`: http://www.oraclebuffer.com/general-discussions/which-index-to-choose-global-or-local-index-for-partitioned-table
