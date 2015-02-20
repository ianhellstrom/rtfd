.. _sql-hints-types-optimizer:
 
Optimizer Hints
---------------
We have already mentioned the ``GATHER_PLAN_STATISTICS`` hint, which can be used to obtain statistics about the execution plan during the execution of a statement.
It is especially helpful when you intend to `diagnose performance issues`_ with a particular statement.
It is definitely not meant to be used in production instances!
 
There is also a ``GATHER_OPTIMIZER_STATISTICS``, which Oracle lists under 'Other hints'.
It can be used to collect bulk-load statistics for CTAS statements and ``INSERT INTO ... SELECT`` statements that use a direct-path insert, which is accomplished with the ``APPEND`` hint, but more on that later.
The opposite, ``NO_GATHER_OPTIMIZER_STATISTICS`` is also provided.
 
The ``OPTIMIZER_FEATURES_ENABLE`` hint can be used to *temporarily* disable certain (newer) optimizer feature after database upgrades.
This hint is typically employed as a short-term solution when a small subset of queries performs badly.
Valid parameter values are `listed in the official documentation`_.

.. _`diagnose performance issues`: http://docs.oracle.com/database/121/ARPLS/d_xplan.htm#ARPLS378
.. _`listed in the official documentation`: http://docs.oracle.com/database/121/REFRN/refrn10141.htm#REFRN10141