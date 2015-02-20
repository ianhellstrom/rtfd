.. _sql-joins-sort-merge:
 
Sort-Merge Join
===============
The sort-merge or simply merge join is actually rarely used by Oracle.
It requires both row sources to be sorted by the join columns from the get-go.
For equality join conditions, the sort-merge join combines both row sources like a zipper: nicely in-sync.
When dealing with ranged-based join predicates, that is everything except ``<>``, Oracle sometimes has to jump a bit back and forth in the probe row source as it strolls through the driving row source, but it pretty much does what a nested loop does: for each row from the driving row source pick the matches from the probe row source.
 
A Cartesian join is basically a sort-merge join, and it shows up as ``MERGE JOIN CARTESIAN`` in the execution plan.
It is Oracle's fall-back plan: if there is no join predicate, then it has no alternative as every row in the driving row source matches each and every row in the probe row source.
What is slightly different for the Cartesian join is that no actual sorting takes place even though the execution plan informs us of a ``BUFFER SORT`` operation.
This operation merely buffers, it does *not* sort.
When the Cartesian product of two row sources is relatively small, the performance should not be too horrendous.
 
A sort-merge join may be performed when there is no index on the join columns, the selectivity of the join columns is low, or the clustering factor is high (i.e. near the number of rows rather than the number of blocks, so the rows are ordered randomly rather than stored in order), so that nested loops are not really an attractive alternative any longer.
Similarly, a sort-merge join may be done instead of a hash join when the hashed row sources are too big to fit in memory.

.. note:: 
   A sort-merge join may spill onto disk too, although that typically is not as bad to performance as with a hash join.
 
When one row source is already sorted and Oracle decides to go ahead with a sort-merge join, the other row source will *always* be sorted even when it is already sorted.
 
The symmetry of the sort-merge join is unique.
In fact, the join order does not make a difference, not even to the performance.