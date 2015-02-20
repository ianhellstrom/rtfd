.. _sql-hints-types-join-operation:
 
Join Operation Hints
--------------------
Join operation hints are also paired:
 
* ``USE_HASH`` / ``NO_USE_HASH``
* ``USE_MERGE`` / ``NO_USE_MERGE``
* ``USE_NL`` / ``NO_USE_NL``
 
These hints allow you to instruct the optimizer to use a hash join, a sort-merge join, or nested loops, respectively.
 
Hash joins support input swapping, which we have discussed when we talked about :ref:`left-deep and right-deep join trees <sql-joins-join-trees>`.
This can be accomplished with ``SWAP_JOIN_INPUTS`` or prohibited with ``NO_SWAP_JOIN_INPUTS``.
 
The left-deep join tree can be enforced with the following hints:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 5-7
 
   /*+ LEADING( t1 t2 t3 t4 )
       USE_HASH( t2 )
       USE_HASH( t3 )
       USE_HASH( t4 )
       NO_SWAP_JOIN_INPUTS( t2 )
       NO_SWAP_JOIN_INPUTS( t3 )
       NO_SWAP_JOIN_INPUTS( t4 ) */
 
We could have also written ``USE_HASH( t4 t3 t2 )`` instead of three separate hints.
 
So, how do we go from a left-deep join ( ( T1 →  T2 ) → T3 ) → T4  to a right-deep join T4 → ( T3 → ( T2 → T1 ) )?
Remember the steps we had to perform, especially the swaps?
The process to go from the left-deep join tree to the right-deep join tree is to swap the order in the following sequence: T4, T3, and T2.
We can thus obtain the right-deep join tree by taking the left-deep join tree as a template and providing the necessary swaps:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 5-7
 
   /*+ LEADING( t1 t2 t3 t4 )
       USE_HASH( t2 )
       USE_HASH( t3 )
       USE_HASH( t4 )
       SWAP_JOIN_INPUTS( t2 )
       SWAP_JOIN_INPUTS( t3 )
       SWAP_JOIN_INPUTS( t4 ) */
       
The ``LEADING`` hint refers to the situation *before* all the swaps.
Important to know is that the left-deep join tree is *always* the `starting point`_.
 
Oracle occasionally bumps into bushy trees when views cannot be merged.
Bushy trees can, however, be practical in what is sometimes referred to as a `snowstorm schema`_, but we shall not go into more details here.
In instances where a bushy join is known to be advantageous you may have to rewrite your query.
For example, you can force Oracle to perform the bushy join ( T1 → T2 ) → ( T3 → T4 ) by writing the query schematically as follows:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 6-14,16-24
 
   SELECT /* LEADING ( v12 v34 )
             USE_HASH( v34 )
             NO_SWAP_JOIN_INPUTS( v34 ) */
     *
   FROM
     (
       SELECT /*+ LEADING( t1 t2 )
                  NO_SWAP_JOIN_INPUTS( t2 )
                  USE_HASH( t2 )
                  NO_MERGE */
         *
       FROM 
         t1 NATURAL JOIN t2
      ) v12
   NATURAL JOIN
     (
       SELECT /*+ LEADING( t3 t4 )
                  NO_SWAP_JOIN_INPUTS( t4 )
                  USE_HASH( t4 )
                  NO_MERGE */
         *
       FROM
         t3 NATURAL JOIN t4
      ) v34
   ;
 
You may have noticed that we have sneaked in the ``NO_MERGE`` hint, which we shall describe in somewhat more detail below.
What is more, we have used a ``NATURAL JOIN`` to save space on the ``ON`` or ``USING`` clauses as they is immaterial to our discussion.
  
Can you force Oracle to do a bushy join without rewriting the query?
 
Unfortunately not.
The reason is that there is no combination of swaps to go from a left-deep join tree to any bushy join tree.
You can do it with a bunch of hints for a zigzag trees, because only some of the inputs are swapped, but bushy trees are a nut too tough to crack with hints alone.
 
When you use ``USE_MERGE`` or ``USE_NL`` it is best to provide the ``LEADING`` hint as well.
The table first listed in ``LEADING`` is generally the driving row source.
The (first) table specified in ``USE_NL`` is used as the probe row source or inner table.
The syntax is the same for the sort-merge join: whichever table is specified (first) is the inner table of the join.
For instance, the combination ``/*+ LEADING( t1 t2 t3 ) USE_NL( t2 t3 ) */`` causes the optimizer to take T1 as the driving row source and use nested loops to join T1 and T2.
Oracle then uses the result set of the join of T1 and T2 as the driving row source for the join with T3.
 
For nested loops there is also the alternative ``USE_NL_WITH_INDEX`` to instruct Oracle to use the specified table as the probe row source and use the specified index as the lookup.
The index key must be applicable to the join predicate.

.. _`starting point`: http://tonyhasler.wordpress.com/2008/12/27/bushy-joins
.. _`snowstorm schema`: http://www.google.com/patents/US20090112793