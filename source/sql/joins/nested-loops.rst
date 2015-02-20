.. _sql-joins-nested-loops:
 
Nested Loops
============
Whenever you have correlated row sources for a left lateral join, Oracle uses nested loops to perform the join.
Nested loops can, however, be used for uncorrelated row sources too, although that often requires some hint trickery, but more on that later when hints are in our focus.
 
Nested loops work by fetching the result from the driving row source and querying the probe row source for each row from the driving row source.
It's basically a nested ``FOR`` loop, hence the name.
The driving row source is sometimes also referred to as the leading row source, which is reflected in the hint ``/*+ leading(...) */`` that can be used to specify the leading object.
 
Nested loops scale linearly: if the row sources double in size, it takes roughly twice as much time to do the join with nested loops, provided that there is a relevant index on the probe row source.
If there is no such index, we lose the linear scalability, because Oracle has to visit each row in the probe row source, which means that in that case nested loops scale quadratically.
Appropriate indexes can often be tricky when the probe source is an inline view; Oracle typically chooses the table, if there is any, as the probe row source.
A somewhat related problem is that the same blocks in the table being probed may be visited many times because different rows are looked at each time.
For small driving row sources the nested loop join is often the best option.
 
Since 11g Oracle can prefetch nested loops, which shows up in the execution plan as the join operation being a child of a table access operation.
This enables the database to first think about what it does to the ROWIDs it obtains from the nested loops.
For instance, if the ROWIDs are all consecutive but not in the buffer cache, the table access operation can benefit from a multi-block read.
 
Multiple nested loops operations can occasionally show up in the execution plan for just one join, which indicates that Oracle used the nested-loop batching optimization technique.
What this method does is transform a single join of two row sources into a join of the driving row source to one copy of the probe row source that is joined to a replica of itself on ROWID; since we now have three row sources, we need at least two nested loops.
The probe row source copy that is used to perform a self join on ROWID is used to filter rows, so it will have a corresponding ``TABLE ACCESS BY ... ROWID`` entry in the execution plan.
This cost-based optimization can often reduce I/O although the execution plan may not transparently display the benefits.
 
Whatever is specified in the ``WHERE`` clause that is exclusively applicable to the driving row source is used by Oracle to filter rows as soon as possible, even though semantically the filter comes after the ``JOIN`` clause.
 
Oracle always uses nested loops for left lateral joins.
What makes lateral joins useful is that predicates derived from columns in the driving row source (i.e. the row source specified *before* the ``LATERAL`` keyword) can be used in the probe row source (i.e. the inline view that follows ``LATERAL``).
 
Beware of the cardinality estimates when you use the ``gather_optimizer_statistics`` hint: for nested loops the estimated row count is *per iteration*, whereas the actual row count is for *all iterations*, as mentioned by Tony Hasler in `Expert Oracle SQL`_ (p. 266).

.. _`Expert Oracle SQL`: http://www.apress.com/9781430259770