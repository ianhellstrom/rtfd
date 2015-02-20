.. _sql-indexes-access-path:
 
Access Paths and Indexes
========================
Let's take another quick look at the access paths we talked about :ref:`earlier <sql-plans-explain-plan>`.
 
The **index unique scan** only returns one row because of a ``UNIQUE`` constraint.
Because of that, Oracle performs only the tree traversal: it goes from the branch node to the relevant leaf node and picks the source row it needs from the table.
 
For an **index range scan** the tree is traversed but Oracle also needs to follow the leaf node chain to find all remaining matching entries. It could be that the next leaf node contains more rows of interest, for instance if you require the maximum value of the current leaf node; because only the maximum value of each leaf block is stored in the branch nodes, it is possible that the current leaf block's maximum index value 'spills over' to the next.
A **table access by index ROWID** often follows an index range scan.
When there are many rows to fetch this combination can become a performance bottleneck, especially when many database blocks are involved.
The cost the optimizer calculates for the table access by index ROWID is strongly influenced by the row count estimates.
 
As the name suggest, a **full index scan** reads the entire index in order.
Similarly, a **fast full index scan** reads the entire index as stored on disk.
It is the counterpart of the **full table access scan**, and it too can benefit from multi-block reads.