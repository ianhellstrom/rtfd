.. _sql-hints-types-join-order:
 
Join Order Hints
----------------
The optimizer lists all join orders to choose the best one.
What it does not do is an exhaustive search.
 
In case you believe a different join order to be useful, you can use one of the join order hints: ``ORDERED`` or ``LEADING``.
The latter is more versatile and should thus be preferred.
 
``ORDERED`` takes no parameters and instructs the optimizer to join the tables in the order as they appear in the ``FROM`` clause.
Because the ``ORDERED`` hint is so basic and you do not want to move around tables in the ``FROM`` clause, Oracle has provided us with the ``LEADING`` hint.
It takes the table names or aliases (if specified) as parameters, separated by spaces.
 
In the optimizer's rock-paper-scissors game, ``ORDERED`` beats ``LEADING`` when both are specified for the same statement.
Moreover, if two or more conflicting ``LEADING`` hints are provided, Oracle ignores all of them.
Similarly, any ``LEADING`` hints are thrown into the bin when they are incompatible with dependencies in the join graph.