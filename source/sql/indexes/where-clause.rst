.. _sql-indexes-where-clause:

Predicates: The ``WHERE`` Clause
================================
The ``WHERE`` clause is the one that determines whether or not indexes can be used efficiently.
One side of each predicate must be as specified in the index(es) for Oracle to be able to use any index.
Whether it is the left-hand side or the right-hand side is irrelevant, although typically it is the left-hand side because SQL is written from the left to the right.
Note that the order sometimes matters though: ``col_name LIKE 'ABC%'`` is not the same as ``'ABC%' LIKE col_name``.
The former searches for ``col_name`` entries that begin with ``ABC``, whereas the latter is the same as the filter ``col_name = 'ABC%'``, that is the ``%`` is not interpreted as a wild card at all.
 
Indexes can only be used when predicates are :term:`sargable <SARGable>`, or search-argument-able, which admittedly is a horrible phrase.
Functions on columns in the index can prohibit index use, particularly when the index is not a function-based index.
Apart from that, some operators are sargable and optimizable (i.e. allow the use of an index): ``=``, ``<``, ``>``, ``>=`` ``IS NULL``; some operators are sargable yet not optimizable: ``<>`` and its equivalents (i.e. ``!=`` and ``^=``) and ``NOT``; and ``LIKE`` with a leading wild card is not sargable and hence not optimizable.
Sargable predicates can be pushed down, which means that a predicate in a statement that references a view or derived table can be 'pushed down' to the view or derived table itself, which avoids having to scan the entire underlying data structure only to filter out a few relevant rows later.
Sargable, non-optimizable predicates can still benefit from the optimizer's efforts; non-sargable predicates cannot though.
 
A SQL statement that links several sargable predicates with an ``OR`` cannot be optimized when the predicates involve different columns.
If, however, the predicates can be rewritten as an equivalent ``IN``-list, which Oracle does internally as a part of its predicate transformations, then Oracle can indeed optimize the statement and therefore utilize existing indexes.
 
Important is, as always, that the data type of each search term matches the data type of the indexed column or expression; it is best that you convert search terms on the right-hand side if necessary but leave the left-hand side as is.
Unnecessary use of ``TO_CHAR()`` and ``TO_NUMBER()`` (on the left-hand side) is not only sloppy but it can hamper index use.
The same goes for ``NVL()`` and the like.
 
If you often encounter fixed expressions or formulas in your predicates, you can create function-based indexes on these expressions.
Make sure that the columns referenced appear in the index in exactly the same way as they appear in the predicates, *and* make sure that the right-hand side does not contain the columns from the index: Oracle does not solve your equations for you.
 
Predicates that are often badly coded include operations on dates.
Yes, it is possible to create a function-based index on ``TRUNC ( expiry_date )`` and use same expression in the database.
However, *all* predicates on the column ``expiry_date`` *must* include ``TRUNC()`` for Oracle to be able to use the index in all cases.
A simple and elegant solution is to provide ranges, either with ``>= TO_DATE(...)`` and ``<= TO_DATE(...)`` or with ``BETWEEN TO_DATE(...) AND TO_DATE``, which is inclusive.
Should you not want it to be inclusive subtract a minimal interval like so: ``TO_DATE(...) - INTERVAL '1' SECOND'``.

Why not the literal ``1/86400`` or ``1/(24*60*60)``?
Well, it may be easy for you to understand something like that because you wrote it (and perhaps added a comment), but it is not always easy to fathom such literals, especially if developers simplify their fractions as in ``7/10800``, which is 56 seconds by the way.
The index may not care about how you write your literals but the other developers in the team do care.
Let the code speak for itself!
 
Since we're on the topic of dates: *never* write ``TO_CHAR ( expiry_date, 'YYYY-MM-DD' ) = '2014-01-01'``.
Leave the ``DATE`` column as is and write ``expiry_date >= TO_DATE ( '2014-01-01','YYYY-MM-DD' )`` and ``expiry_date < TO_DATE ( '2014-01-01','YYYY-MM-DD' ) + INTERVAL '1' DAY`` instead. [#interval]_
Yes, it's a bit more typing, but that way an index range scan can be performed and you do not need a function-based index.
 
'But what if I need only products from the fridge that expire in February?'
Since repetition is the mother of learning, here comes: specify ranges from the first day of February to the last day of February.
 
'But I want to show the total number of products by the year and month of the expiry date.'
You could use the ``EXTRACT ( YEAR FROM expiry_date )`` and similarly for the month, ``TRUNC( expiry_date, 'MM' )`` or ``TO_CHAR ( expiry_date, 'YYYY-MM' )``.
However, since you are pulling in all data from the table, a full table scan really is your best option.
Yes, you read that right: a full table scan is the best alternative; we'll say more about full table scans in a few moments.
Furthermore, if you already have an index on ``expiry_date`` and it is stored in order (i.e. it is not a ``HASH`` index on a partitioned table), then the ``GROUP BY`` can make use of the index without any additional function-based indexes.
 
The ``LIKE`` comparison operator is also often a cause for performance problems because applications tend to allow wild cards in strings, which means that a search condition à la ``WHERE col_name LIKE '%SOMETHING%'`` is not uncommon.
Obviously, you cannot create a sensible index for a predicate like that.
It is tantamount to asking a dictionary to provide you with a list of all possible sequences of characters in any position.
 
The ``INDEX`` hint, as described by `Laurent Schneider`_, is — contrary to what is claimed by the said author — *not* always beneficial for predicates with leading and trailing wild cards, so be sure to try it out.
An index is, however, used when such a predicate is specified with bind variables:
 
.. code-block:: sql
   :linenos:
 
    VARIABLE l_like VARCHAR2(20);
    EXEC :l_like := '%SOMETHING%';
 
    SELECT
      *
    FROM
      tab_name
    WHERE
      col_name LIKE :l_like;
 
If you always look for things *ending* with a series of characters, such as ``LIKE '%ABC'`` you *can* use an index.
Just create the index on ``REVERSE ( col_name )`` and reverse the string you are looking for itself, and voilà, it works: ``WHERE REVERSE ( col_name ) LIKE 'CBA%'``.
 
To search in a case-insensitive manner you have to create a function-based index, say, ``UPPER(col_name)``.
You could have gone with ``LOWER(col_name)`` and whatever you prefer is really up to you.
All that matters is that you are thrifty and consistent: switching back and forth between ``UPPER()`` and ``LOWER()`` is a bad idea because the database has to maintain two indexes instead of one, and you really only need one index.
Which function you choose for case-insensitive searches is irrelevant but document whichever you choose, so it is clear to all developers on the team.
 
In an international setting you may want to use ``NLS_UPPER( col_name, 'NLS_SORT = ...' )``, so that for instance — for ``... = XGERMAN`` — ``ß`` and ``ss`` are seen as equivalent.
The parameters ``NLS_SORT`` and ``NLS_COMP`` can be made case- or accent-insensitive by appending ``_CI`` or ``_AI`` to their `sort name values`_ respectively.
The ``NLS_SORT`` parameter can be used to alter a session or the entire system.
 
For purely linguistic rather than binary searches of text, you can set the system's or session's ``NLS_COMP = LINGUISTIC``.
The performance of linguistic indexes can thus be improved: ``CREATE INDEX ix_col_name_ling on tab_name ( NLSSORT( col_name, 'NLS_SORT = FRENCH' ) )``, for French for example.
 
We have already seen that with function-based indexes it is important to have the exact same expression save for irrelevant spaces.
A functionally equivalent expression that is syntactically different prohibits the use of an index, so writing ``REGEXP_LIKE()`` in your ``WHERE`` clause when you have used ``INSTR()`` in the index means that the optimizer will ignore the index.
 
For Oracle Database 11g there is a good book on `expert indexing`_, if you want to learn more about indexes.


.. _`Laurent Schneider`: http://laurentschneider.com/wordpress/2009/07/how-to-tune-where-name-likebc.html
.. _`sort name values`: http://docs.oracle.com/database/121/NLSPG/applocaledata.htm#NLSPG593
.. _`expert indexing`: http://www.apress.com/9781430237358

.. rubric:: Notes

.. [#interval] The ``INTERVAL`` function has one major disadvantage: ``SELECT TO_DATE ( '2014-01-31', 'YYYY-MM-DD' ) + INTERVAL '1' MONTH FROM dual`` leads ``ORA-01839: date not valid for month specified`` error. The function ``ADD_MONTHS()`` solves that problem.