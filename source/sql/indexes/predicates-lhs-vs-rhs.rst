.. _sql-indexes-lhs-vs-rhs:

Predicates: LHS vs RHS
======================
They way you *syntactically* write predicates matters, even though *logically* various forms of predicates are equal.
The difference between the left-hand side (LHS) and the right-hand side (RHS) of equality and inequality predicates is significant.
Oracle only evaluates the right-hand side of predicates for the compilation, and the left-hand side should ideally refer to indexed columns as they appear in the index.
 
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