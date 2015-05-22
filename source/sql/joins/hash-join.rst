.. _sql-joins-hash:
 
Hash Join
=========
In a hash join, Oracle hashes the join key of the 'driving' row source in memory, after which it runs through the 'probe' row source and applies the hash to obtain the matches.
We have placed the words 'driving' and 'probe' in quotes to indicate that the nomenclature is slightly different for hash joins though still applicable.
Because of the hashing it is clear that an index on the probe row source will not improve the performance of the hash join.
The only indexes that are beneficial in a hash join are indexes on predicates in the ``WHERE`` clause, but that is — as we have said — not specific to hash joins at all.
Moreover, when the probe row source is table, a hash join does not visit blocks multiple times, since the database goes through the probe row source once.
In fact, if Oracle decides to do a full table scan on the probe row source it may also decide to do multi-block reads to bump its retrieval efficiency.
 
It's not all peachy though.
Suppose that the probe row source is huge but only very few rows match the join clause and that we have no predicate or one that is barely selective.
With a hash join he database visits many blocks that contain no data we're interested in, because we cannot retrieve the rows through an index.
Whether the balance is tipped in favour of nested loops with an index lookup of the probe row source, if of course available, that perhaps visits blocks over and over again, or a hash join with a single scan of each block depends mainly on the selectivity of the join condition and the clustering factor of the index.
In these cases it is often advantageous to fix the execution plan with hints once you have discovered and argued that one consistently outperforms the other.
 
Hash joins scale linearly too, but there is one caveat — isn't there always?
The entire hash table has to fit in memory.
If it does not, the hash table will spill onto disk, which ruins the linear scalability to the ground.
As always, select only the columns you need as the size of the hash table may increase dramatically and thus ruin the performance benefit when Oracle runs out of memory.
 
Another gotcha with hash joins is that they can only be used with equality join conditions.

.. _sql-joins-join-trees:

Join Orders and Join Trees
--------------------------
When you hash-join several row sources with an inner join, Oracle can in principle swap the order without affecting the result set.
Why would it want to do that?
Well, the optimizer may discover that one particular join order is better than all the others.
For an inner join of :math:`n` tables, there are :math:`n!` possible join orders.
For four tables, we have :math:`4! = 4\cdot 3\cdot 2\cdot 1 = 24` possibilities.
So the chances are that there is at least one that is significantly better and one that is the absolute worst.
 
Let's take four tables: T1, T2, T3, and T4.
A so-called `left-deep join tree`_ is obtained in the following way:
 
#. Place T1's hash cluster in a workarea.
#. *Join T1 and T2. Call the intermediate result set J12.*
#. Place J12's hash cluster in a workarea.
#. Drop T1's workarea.
#. *Join J12 and T3. Call the intermediate result set J123.*
#. Place J123's hash cluster in a workarea.
#. Drop J12's workarea.
#. *Join J123 and T4.*
#. Drop J123's workarea.
 
The italicized items are the actual logical steps in the join order.
The left row source is always the driving row source in our notation, and the right row source is always the probe row source.
We can also write this succinctly as ( ( T1 → T2 ) → T3 ) → T4.
 
For a right-deep join tree we have the following steps:
 
#. Place T4's hash cluster in a workarea.
#. Place T3's hash cluster in a workarea.
#. Place T2's hash cluster in a workarea.
#. *Join T2 and T1. Call the intermediate result set J21.*
#. Place J21's hash cluster in a workarea.
#. Drop T2's workarea.
#. *Join T3 and J21. Call the intermediate result set J321.*
#. Place J321's hash cluster in a workarea.
#. Drop T3's workarea.
#. *Join T4 and J321.*
#. Drop all remaining workareas.
 
We can write this as T4 → ( T3 → ( T2 → T1 ) ).
 
What is hopefully clear from these sequences is that a left-deep join tree requires two concurrent workareas, whereas a right-deep join tree has as many workareas as row row sources.
So, why on earth do we ever want a right-deep join tree?
 
Suppose for a second that T1 is enormous and the remaining tables are relatively small, which often happens in data warehouses.
Just think of T1 as being the fact table (e.g. sales) and T2, T3, and T4 dimension tables (e.g. products, customers, and suppliers).
In a left-deep join tree we would create a large workarea with T1, and potentially do a couple of Cartesian joins on the dimension tables as these often do not have join conditions with one another.
This would leave us with a monstrous hash cluster for T1 that will likely not fit in memory.
Moreover, the hash clusters of the Cartesian joins of the dimension tables may also easily be more than Oracle can handle.
The right-deep join tree places the smaller tables in workareas and finally scans the large table T1 instead.
In doing so, we have more workareas but they are all likely to fit in memory, thus allowing us to feel the wind of linear scalability in our hair as we speed through the joins.
 
Let's not get carried away now.
How do we obtain a right-deep from a left-deep join tree?
We can go from a left-deep join tree to a right-deep join tree in the following manner:
 
#. Swap T4: T4 → ( ( T1 → T2 ) → T3 ).
#. Swap T3: T4 → ( T3 → ( T1 → T2 ) ).
#. Swap T2: T4 → ( T3 → ( T2 → T1 ) ).
 
Notice that in the first swap we have obtained an intermediate result set as a probe row source.
 
The corresponding (simplified) execution plans would look something like the ones shown below.
In particular, for the left-deep join tree we have:
 
.. code-block:: none
 
    -------------------------------------
    | Id | Operation             | Name |
    -------------------------------------
    | 0  | SELECT STATEMENT      |      |
    | 1  |  HASH JOIN            |      |
    | 2  |   HASH JOIN           |      |
    | 3  |    HASH JOIN          |      |
    | 4  |     TABLE ACCESS FULL |  T1  |
    | 5  |     TABLE ACCESS FULL |  T2  |
    | 6  |    TABLE ACCESS FULL  |  T3  |
    | 7  |   TABLE ACCESS FULL   |  T4  |
 
And for the right-deep join tree we see:
 
.. code-block::none
 
    -------------------------------------
    | Id | Operation             | Name |
    -------------------------------------
    | 0  | SELECT STATEMENT      |      |
    | 1  |  HASH JOIN            |      |
    | 2  |   TABLE ACCESS FULL   |  T4  |
    | 3  |   HASH JOIN           |      |
    | 4  |    TABLE ACCESS FULL  |  T3  |
    | 5  |    HASH JOIN          |      |
    | 6  |     TABLE ACCESS FULL |  T2  |
    | 7  |     TABLE ACCESS FULL |  T1  |
 
These are of course not all of Oracle's options.
`Bushy joins`_ (yes, they are really called that) or zigzag join trees have some of the row sources swapped but not all as in the case of left-deep and right-deep join trees.
An example of such a zigzag tree would be the following: ( T1 → T2 ) → ( T3 → T4 ).
To be specific, we obtain that particular join order as indicated:
 
#. Join T1 and T2. Call the intermediate result set J12.
#. Join T3 and T4. Call the intermediate result set J34.
#. Join J12 and J34.
 
Interestingly, bushy joins are *never* considered by the optimizer.
Hence, if you believe a bushy join to be the best join order possible, you have to force Oracle with the ``leading`` hint.

.. _sql-join-partitions:

Partitioned Hash Joins
----------------------
For two tables that are equijoined and both partitioned identically, Oracle does a `full partition-wise join`_, which shows up as a ``PARTITION HASH`` parent operation to the ``HASH JOIN`` in the execution plan.
Similarly it can pop up in a parallel SQL statement as ``PX PARTITION HASH``.
Each parallel query server reads data from a particular partition of the first table and joins it with the appropriate rows from the corresponding partition of the second table.
Query servers have no need to communicate to one another, which is ideal.
The only downside is if there is at least one partition that is significantly larger than all the others, as this may affect the balancing of the load.
 
When only one table is partitioned, Oracle can go with a (parallel) `partial partition-wise join`_.
It (re)partitions the other table on the fly based on the partitioning scheme of the reference table.
Once the partitioning is out of the way, the database proceeds as it does with a full partition-wise join.
 
It is `generally recommended`_ to use hash instead of range partitioning for partition-wise joins to be effective, mainly because of possible data skew that leads to some partitions being larger than others.
Furthermore, `the number of partitions in relation to the DOP`_ is relevant to the performance.
Ideally, the number of partitions is a multiple of the number of query servers.
 
Both hash and sort-merge joins are possible for full partition-wise joins.
More details can be found in the excellent book `Expert Oracle SQL`_.

.. _`Bushy joins`: http://tonyhasler.wordpress.com/2008/12/27/bushy-joins
.. _`full partition-wise join`: http://docs.oracle.com/database/121/VLDBG/to_vldbg1348_d471.htm
.. _`partial partition-wise join`: http://docs.oracle.com/database/121/VLDBG/to_vldbg1349_d472.htm
.. _`the number of partitions in relation to the DOP`: http://blogs.oracle.com/datawarehousing/entry/partition_wise_joins
.. _`generally recommended`: http://docs.oracle.com/database/121/VLDBG/to_vldbg1353_d476.htm
.. _`Expert Oracle SQL`: http://www.apress.com/9781430259770
.. _`left-deep join tree`: http://www.oaktable.net/content/right-deep-left-deep-and-bushy-joins
