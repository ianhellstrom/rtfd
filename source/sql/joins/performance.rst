.. _sql-joins-performance:
 
Join Performance: ``ON`` vs ``WHERE``
=====================================
Now that we are equipped with a better appreciation and understanding of the intricacies of the various join methods, let's revisit the queries from the introduction.
 
Queries 1a and 1b are logically the same and Oracle will treat them that way.
First, let's assume there there is an index on ``department_id`` in both tables.
Such an index is only beneficial to nested loops because that particular column is in the join clause.
As such, the ``employees`` table is likely to become the driving row source, for a filter like ``LIKE last_name = 'X%'`` is probably very selective in many instances, which means that the number of iterations will be relatively low.
While accessing the ``employees`` table, Oracle will apply the filter because it knows that single-column join conditions in the ``ON`` clause of inner joins are the same as predicates in the ``WHERE`` clause.
The database will do so either with a lookup if the relevant index on ``employees`` is selective enough or by means of a full table scan if it is not highly selective.
It will then use the index on ``departments`` to access its data by ROWID, thereby joining it to the data from the leading row source.
When the filter on ``last_name`` is not as selective, especially when the cardinality of the ``departments`` table is lower than the cardinality of the ``employees`` table *after* the filter has been applied, the roles of driving and probe row sources are reversed.
 
If no such indexes exist at all, then a hash join seems logical.
Whether the ``departments`` or ``employees`` table is used to generate an in-memory hash cluster depends on what table Oracle believes will be best based on the cardinality estimates available.
We basically have the same logic as before: for highly selective filters, Oracle will use the ``employees`` table as the driving row source, otherwise it will pick (on) the ``departments`` table to take the lead.
 
Queries 2 and 3 yield different result sets, so it's more or less comparing apples and oranges.
Nevertheless, with an appropriate, selective index on ``last_name`` Oracle will probably settle for nested loops for Query 2 (i.e. the one with the ``WHERE`` clause), and a hash join for Query 3 (i.e. the one with the ``ON`` clause).
If the index on ``last_name`` is not selective at all and its clustering factor is closer to the number of rows than the number of blocks, then Query 2 may also be executed with a hash join, as we have discussed earlier.
Should the SQL engine decide on nested loops for Query 3, it is to be expected that the ``departments`` table be promoted to the position of driving row source because Oracle can use the single-column join condition on ``last_name`` as an access predicate.
 
Please note that a sort-merge join is possible in all instances.
The sort-merge join is rarely Oracle's first choice when faced with equality join conditions, especially when the tables involved are not sorted to start with.
 
So, what you should take away from this section is that even though the ``WHERE`` clause is technically a *post*-join filter, it can be and often is used by Oracle when it fetches the data of the leading row source, analogously to single-column predicates specified in the ``ON`` clause, thereby reducing the number of main iterations (i.e. over the driving row source) or the number of index lookups (in the probe row source) for nested loops, or the size of the in-memory hash cluster for a hash join.
For sort-merge joins these predicates can be used to minimize the size of the tables to be sorted, if one or both tables require reordering.