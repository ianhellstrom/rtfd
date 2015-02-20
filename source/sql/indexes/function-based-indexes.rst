.. _sql-indexes-fbi:
 
Function-Based Indexes and NULLs
================================
By default Oracle does not store null rows in a (B-tree) index.
You can add them with a simple trick:
 
.. code-block:: sql
   :linenos:
 
   CREATE INDEX index_name
     ON tab_name ( nullable_col_name, 1 );
 
The 'trick' is of course nothing but a function-based index.
By adding nulls to your (function-based) index, you ensure that Oracle is able to avoid full table scans when you ask for ``col_name IS NULL`` .
Alternatively, you can use ``NVL`` as a function in your index if you want to; you have to remember that your index can only be used if you use the same function in your filter predicates.
 
That is a common thread in function-based indexes though: you have to have the exact same predicate as used in the index for the index to be used.
Oracle has no compiler that evaluates and simplifies (mathematical) expressions, so a ``WHERE`` clause like ``WHERE col_a + col_b = 42`` does not use an index on ``col_a`` because the lef-hand side also includes ``col_b``.
To use an index on ``col_a`` , you have to rewrite the predicate as ``WHERE col_a = 42 - col_b``.
Similarly, ``LN ( EXP(col_real) )`` is not simplified to ``col_real`` for ``col_real`` a column of real-valued numbers.
Oracle is smart but you cannot expect it to do everything for you: not even state-of-the-art computer algebra systems like Mathematica and Maple can simplify all crazy expressions you can think of.
 
The power of function-based indexes lies in the fact that often your applications have to filter for bits and pieces of data that are already available but normal indexes cannot be used, which often happens because of conversion, mathematical, and string-manipulation functions, in particular ``SUBSTR()`` and  ``LOWER()`` or ``UPPER()``.
Suppose you have a sudden, inexplicable urge to behave like a business analyst and you want to generate a report of the average temperature of all products with an expiry date of products in your fridge for a particular ISO workweek; if you think this is an odd request then please replace temperature with amount, expiry date with the date of the transaction, and the fridge with a sales table.

You create the following function-based index: ``CREATE INDEX ix_workweek ON fridge ( TO_CHAR(expiry_date, 'IW') )``.
If you now use a clause like ``WHERE TO_CHAR(expiry_date, 'IW') = '20'``, you can see all products with an expiry date in workweek twenty using the index ``ix_workweek``; the single quotes in the ``WHERE`` clause are included because the resulting expression is of type ``VARCHAR2``.
Avoid implicit conversions as much as possible; not because of the almost negligible conversion performance penalty but because you rely on Oracle to do it right in the background *and* it is considered bad style!
 
Imagine you have created a function-based index on a certain concatenation of columns, for instance ``manufacturer || '''s ' || product``, then you can use that exact expression in the ``WHERE`` clause.
This is, however, an extremely fragile and overly complex solution that does not really belong in your queries.
Such logic is usually application-specific, so it should either be handled by the application itself or a layer of PL/SQL between the database and your user interface that extracts the data and then with the aid of an auxiliary function formats it correctly: it is always a good idea to separate the data-access layer from the application layer, as it minimizes the number of places you have to search and replace something whenever a change requests ends up on your desk.
 
Why?
What if next week someone decides to search for ``WHERE product || ' by '  || manufacturer = ...`` instead?
You then need to change not only your query but also your index.
Worse still, what if you want to list only stuff from one manufacturer?
You can't even use the index!
Why make life hard when you can just add both ``manufacturer`` and ``product`` to your index and search for each one individually, separated by a beautiful ``AND``?!
If at this point you think that no one is that thick, then I'll just say that if I'd have had a dollar for each time I saw something similar (usually with first and last names), I'd be rich.
And if I'd have had an extra dollar for each time people complained about shitty performance because of such an abomination of a predicate and demanded an index to solve it, I'd be stinking rich.
 
By the way, we're not done with nulls yet.
Queries can sometimes run `without utilizing an index`_ because a ``NOT NULL`` constraint is absent.
Constrains are thus not only important to enforce consistency but also to ensure consistent performance of your queries.
Furthermore, functions on columns *with* ``NOT NULL`` constraints can lead to the same (unwanted) behaviour.
The reason is that Oracle does not know whether a function preserves the ``NOT NULL`` constraint of the column.
For some internal functions, though, Oracle knows that ``NOT NULL`` is preserved, which means that it can still use any available and relevant indexes.
Examples of such internal functions are ``LOWER()`` and ``UPPER()``.
 
User-defined functions are black boxes as far as the optimizer is concerned.
As of Oracle Database 11g, `virtual columns`_ can be used to circumvent the issue.
Virtual columns are not stored and they are derived (or computed) from other columns in the same table.
They are created like normal columns but with the syntax of ``col_name [ data_type ] [ GENERATED ALWAYS ] AS ( expression ) [ VIRTUAL ]``, where the entries between square brackets are optional although highly recommended to indicate that the column in question is merely virtual.
An index on a virtual column is like a function-based index on a normal column, but it has the benefit that you can add the ``NOT NULL`` constraint to it.
Hence, the optimizer can treat the expression as a ``NOT NULL``-preserving function.
Sweet!

.. _`without utilizing an index`: http://use-the-index-luke.com/sql/where-clause/null/not-null-constraint
.. _`virtual columns`: http://www.oracle-base.com/articles/11g/virtual-columns-11gr1.php