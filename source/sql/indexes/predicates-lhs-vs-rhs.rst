.. _sql-indexes-lhs-vs-rhs:

Predicates: LHS vs RHS
======================
They way you *syntactically* write predicates matters, even though *logically* various forms of predicates are equal.
The difference between the left-hand side (LHS) and the right-hand side (RHS) of equality and inequality predicates is significant.
Oracle transforms a predicate such as :math:`A \star B`, with :math:`\star` any valid operator (e.g. ``=``, ``>``, or ``LIKE``) to a canonical form, where typically the LHS contains only the (indexed) column name.
The most this transformation does is swap the operands.
The exception to this rule is when both sides of the operator are non-standard, for instance a function is applied or numbers are added.
That Oracle transforms the predicate ``SYSDATE - 1 <= col_name`` to ``col_name >= SYSDATE - 1`` can easily be verified by looking at the execution plan.
When it encounters something like ``col_name + 2 = 4*another_col_name``, it is not smart enough to rewrite this as ``col_name = 4*another_col_name - 2``, so you have to do it for Oracle.
 
What about predicates that emulate full-text searches like ``WHERE col_name LIKE '%something interesting%'``?
Short answer: you're pretty much screwed.
Standard indexes are not designed to meet that requirement.
It's like asking you to search for a book with an ISBN that has 123 somewhere in it.
Good luck!
Long answer: `Oracle Text`_.
Yes, it's the long answer, even though it's only two words, because it requires you to do some digging.
Oracle Text comes with all editions of the database but it's beyond our scope.
With it you can use SQL to do full-text searches, which is especially interesting you need to mine texts; it's overkill for occasional queries with a non-leading ``LIKE`` though.

.. _`Oracle Text`: http://www.oracle.com/technetwork/database/enterprise-edition/index-098492.html
