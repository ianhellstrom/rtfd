.. _sql-hints-types-goals:
 
Optimization Goals and Approaches
---------------------------------
Oracle only lists two hints in this category: ``ALL_ROWS`` and ``FIRST_ROWS( number_of_rows )``.
These are mutually exclusive.
If you happen to be drunk while programming and inadvertently write both hints in the same statement, Oracle will go with ``ALL_ROWS``.
 
In mathematical optimization nomenclature, these two hints affect the objective function.
``ALL_ROWS`` causes Oracle to optimize a statement for throughput, which is the minimum *total* resource consumption.
The ``FIRST_ROWS`` hint does not care about the throughput and instead chooses the execution plan that yields the first ``number_of_rows`` specified as quickly as possible.
 
.. note::
   Oracle ignores ``FIRST_ROWS`` in all ``DELETE`` and ``UPDATE`` statements and in ``SELECT`` statement blocks that include sorts and/or groupings, as it needs to fetch all relevant data anyway.