.. _sql-hints-types-access:
 
Access Path Hints
-----------------
Access path hints determine how Oracle accesses the data you require.
They can be divided into two groups: access path hints for tables and access path hints for indexes.
 
Tables
^^^^^^
The most prominent hint in this group is the ``FULL( tab_name )`` hint.
It instructs the optimizer to access a table by means of a full table scan.
If the table you want Oracle to access with a full table scan has an alias in the SQL statement, you have to use the alias rather than the table name (without the schema name) as the parameter to ``FULL``.
For named query blocks you have to provide the query block's name as discussed previously.
 
In this group are also the ``CLUSTER`` and ``HASH`` hints, but they apply only to tables in an indexed cluster and hash clusters respectively.
 
Indexes
^^^^^^^
The hints in this group all come in pairs:
 
* ``INDEX`` / ``NO_INDEX``
* ``INDEX_ASC`` / ``INDEX_DESC``
* ``INDEX_FFS`` / ``NO_INDEX_FFS``
* ``INDEX_SS`` / ``NO_INDEX_SS``
* ``INDEX_SS_ASC`` / ``INDEX_SS_DESC``
* ``INDEX_COMBINE`` / ``INDEX_JOIN``
 
All these hints take at least one parameter: the table name or alias in the SQL statement.
A second parameter, the index name(s), is optional but often provided.
If more than one index is provided, the indexes are separated by at least one space; the ``INDEX_COMBINE`` hint is recommended for this use case though.
 
Let's get cracking.
The first pair instructs the optimizer to either use (or not use) an index scan on a particular table.
If a particular index is specified, then Oracle uses that index to scan the table.
If no index is specified and the table has more than one index, the optimizer picks the index that leads to the lowest cost when scanning the data.
These hints are valid for any function-based, domain, B-tree, bitmap, and bitmap join index.
 
Similarly, you can tell the optimizer that it needs to scan the specified index in ascending order with ``INDEX_ASC`` or descending order with ``INDEX_DESC`` for statements that use an index range scan.
Note that if your index is already in descending order, Oracle ignores the ``INDEX_DESC`` hint.
 
No, ``FFS`` does not stand for "for f*ck's sake".
Instead it indicates that Oracle use a fast full index scan instead of a full table scan.
 
An index skip scan can be enabled (disabled) with ``INDEX_SS`` (``NO_INDEX_SS``).
For index range scans, Oracle scans index entries in ascending order if the index is in ascending order and in descending order if the index is in descending order.
You can override the default scan order with the ``INDEX_SS_ASC`` and ``INDEX_SS_DESC`` hints.
 
The pair ``INDEX_COMBINE`` and ``INDEX_JOIN`` is the odd one out, as they are not each other's opposites.
``INDEX_COMBINE`` causes the optimizer to use a bitmap access path for the table specified as its parameter.
If no indexes are provided, the optimizer chooses whatever combination of indexes has the lowest cost for the table.
When the ``WHERE`` clause of a query contains several predicates that are covered by different bitmap indexes, this hint may provide superior performance, as bitmap indexes can be combined very efficiently.
If the indexes are not already bitmap indexes, Oracle will perform a ``BITMAP CONVERSION`` operation.
As Jonathan Lewis puts it in the comments section of `this blog post`_: it's a damage-control access path.
You generally would not want to rely on bitmap conversions to combine indexes; it is often much better to improve upon the index structure itself.
 
The ``INDEX_JOIN`` instructs the optimizer to join indexes (with a hash join) to access the data in the table specified.
You can only benefit from this hint when there is a *sufficiently* small number of indexes that contains all columns required to resolve the query.
Here, 'sufficiently' is Oraclespeak for as few as possible.
This hint is worth considering when your table has `many indexed columns but only few of them are referenced`_ (p. 560) in your statement.
In the unfortunate event that Oracle decides to join indexes and you are certain that that is not the optimal access path, you cannot directly disable it.
Instead you can use the ``INDEX`` hint with only one index or the ``FULL`` hint to perform a full table scan.

.. _`many indexed columns but only few of them are referenced`: http://www.apress.com/9781430257585
.. _`this blog post`: http://jonathanlewis.wordpress.com/2007/02/08/index-combine