.. _sql-subqueries-scalar:
 
Scalar Subqueries
=================
Let's start with the conceptually easiest type of the subqueries: scalar subqueries.
Usually a scalar subquery in the ``SELECT`` needs to be evaluated for each row of the outer query (i.e. the part without the scalar subquery).
If the scalar subquery calculates, say, a sum of a certain values from a large table, this means that the sum has to be calculated many times.
Since it makes no sense to scan the same table over and over again for a value that really only needs to be computed once, Oracle Database 12c has the ability to unnest the scalar subquery, which means that it can convert the scalar subquery into, for instance, a ``GROUP BY`` view that is (outer-)joined to the table without the scalar subquery.
 
`Tanel Poder`_ has written about this feature, and his advice on the matter is spot-on too: you rarely need scalar subqueries in the ``SELECT`` clause and rewriting your scalar subqueries as joins opens up a whole collection of optimization options that Oracle can tinker with in the background.
Oracle has had the ability to unnest scalar subqueries prior to 12c, but there was still a `bug associated with scalar subquery unnesting`_ until 11.2.

.. _`Tanel Poder`: http://blog.tanelpoder.com/2013/08/13/oracle-12c-scalar-subquery-unnesting-transformation
.. _`bug associated with scalar subquery unnesting`: http://timurakhmadeev.wordpress.com/2011/06/28/scalar-subquery-unnesting