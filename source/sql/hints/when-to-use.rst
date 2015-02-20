.. _sql-hints-when:
 
When To Use Hints
=================
Oracle recommends that "hints [...] be used sparingly, and only *after you have collected statistics on the relevant tables and evaluated the optimizer plan without hints* using the ``EXPLAIN PLAN`` statement."
We have added the emphasis because that specific phrase is critical: don't add hints because you *think* you know better than the optimizer.
Unless you are an Oracle virtuoso you probably do not understand the fine print of each hint and its impact on the performance of your statements -- we certainly don't.
 
Still, you may have good reasons to add hints to your statements.
We have listed some of these reasons below.
 
* For demonstrative purposes.
* To try out different execution plans in the development phase.
* To provide Oracle with pertinent information when no such information is available in statistics.
* To fix the execution plan when you are absolutely sure that the hints lead to a significant performance boost.
 
There may be more, but these four sum up the whys and wherefores pretty well.