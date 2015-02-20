.. _sql-subqueries:

**********
Subqueries
**********
`Subqueries`_ come in different shapes and forms: scalar subqueries, nested (single- or multi-row) subqueries (in the ``WHERE`` clause), correlated subqueries, inline views (in the ``FROM`` clause), and factored subqueries (in the ``WITH`` clause), which are also known as common table expressions (CTEs) outside Oracleland.
They are pretty versatile constructs as they can appear in almost any clause of DML statements with the exception of the ``GROUP BY`` clause.
 
Many of the subquery types are interchangeable.
Inline subqueries can be rewritten as factored subqueries, and factored subqueries can often be rewritten as inline subqueries; recursive factored subqueries are the exception to this rule.
Recursive factored subqueries can, nevertheless, typically be written as hierarchical queries using the ``CONNECT BY ... START WITH ...`` syntax.
Similarly, you can write a correlated subquery as a join, but not every join can become a correlated subquery.

.. _`Subqueries`: http://docs.oracle.com/database/121/SQLRF/queries007.htm#SQLRF52357

.. toctree::
   :hidden:

   Scalar Subqueries <scalar-subqueries>
   Nested and Correlated Subqueries <nested-subqueries>
   Subquery Unnesting <subquery-unnesting>
   Inline Views and Factored Subqueries <inline-views-ctes>