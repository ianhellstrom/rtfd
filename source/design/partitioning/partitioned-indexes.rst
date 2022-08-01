.. _model-partition-index:

Partitioned Indexes
===================
Indexes of partitioned tables can be partitioned too.
A **local partitioned index** is an index of a partitioned table that is partitioned on the same columns.
Each index partition is associated with one table partition, hence the name 'local'.
OLAP environments are prime candidates for local partitioned indexes.
When data in a certain partition is updated or deleted, only the data in a single index partition is affected; only index partitions have to be rebuilt or maintained.
Local partitioned index are intimately tied to the table partitions to which they belong: you cannot add or remove an index partition.
Instead, you have to modify the table itself; the index will follow.

Note that bitmap indexes can *only* be local partitioned indexes of a partitioned table.
Global bitmap indexes are only available to non-partitioned tables.

Local partitioned indexes can be either **local prefixed indexes** or **local non-prefixed indexes**.
When the partition key is on the leading edge of the indexed column list we have a local prefixed index.
Queries with predicates on the index can always use partition elimination or :ref:`partition pruning <sql-partition-pruning>`, which reduces I/O.
Local non-prefixed index cannot in all instances eliminate partitions as the partition key is not on the leading edge of the index.

Non-prefixed indexes are often a good idea for historical information.
Let's go back to our classical fridge example and the household robot.
The fridge contains all products that your robot can retrieve at any given time.
You now want to program the robot to to throw away all products that have been in the fridge for more than a year.
However, you also want fast access to your produce based on the product.
What you can do is range-partition your fridge based on the purchase date, so that you can drop the partition with the oldest stuff independently of the others.
With a non-prefixed index on the product identifier you can benefit from a quick lookup.
To support the fast dropping of the oldest partition and its associated index, you can partition the non-prefixed index on the purchase date too.

If that example sounds too weird for you, replace fridge with sales, purchase date with sales date, and product with account number.
That ought to do it!

A **global partitioned index** is a B-tree index that is partitioned independently of its table.
Global partitioned indexes are ideal for OLTP systems, where :term:`IUD <IUD statements>` performance is critical.
It should be obvious that a global partitioned index is harder to maintain as it may span many partitions.
For example, when a single table partition is dropped, the entire global index is affected.
What is more, a global index must be recreated in its entirety even when only a single partition is recovered to a point in time; a local index does not have this issue.
Global partitioned indexes can only be prefixed.

Oracle will automatically use multiple parallel processes for `fast full index scans and multi-partition range scans`_, so that each process reads a local index partition.
This does not apply to SQL statements that probe individual rows.

Please note that an index on a partitioned table may be global and non-partitioned.
Such indexes can become an issue when data a partition is truncated and the index becomes `unusable`_.
Rebuilding such a global non-partitioned index can take a long time when the underlying table is huge.
Global non-partitioned indexes are most common in data warehousing environments to enforce primary key constraints.

.. _`fast full index scans and multi-partition range scans`: http://www.dba-oracle.com/t_global_vs_local_partitioned_indexes.htm
.. _`unusable`: http://www.databasejournal.com/features/oracle/article.php/3682121/Partition-Pitfalls-in-Oracle.htm
