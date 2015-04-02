.. _plsql-bind-sharing:

Adaptive Cursor Sharing and Adaptive Execution Plans
====================================================
That's not the end of the story though.
As of 11g, Oracle has introduced the concept of adaptive cursor sharing.
Based on the performance of a SQL statement, the execution plan may be marked for revision the next time the statement is executed, even when the underlying statistics have not changed at all.
 
In ``v$sql`` this is indicated by the columns ``is_bind_sensitive`` and ``is_bind_aware``.
The former indicates that a particular ``sql_id`` is a candidate for adaptive cursor sharing, whereas the latter means that Oracle acts on the information it has gathered about the cursor and alters the execution plan.
 
Problematic is that adaptive cursor sharing can only lead to an improved plan *after* the SQL statement has `tanked at least once`_.
You can bypass the initial monitoring by supplying the ``BIND_AWARE`` hint: it instructs the database that the query is bind sensitive and adaptive cursor sharing should be used from the very first execution onwards.
A prerequisite for the hint to be used is that the bind variables only appear in the ``WHERE`` clause and an applicable histogram is available.
The hint may improve the performance but you should be aware that it's rarely the answer in the case of :ref:`generic static statements <plsql-bind-smart-logic>`, which we describe below.
The ``NO_BIND_AWARE`` hint does exactly the opposite: it disables bind-aware cursor sharing.
 
Frequency histograms are important to adaptive cursor sharing.
The problem is that they are `expensive to compute, unstable when sampled, and the statistics have to be collected at the right time`_.
In Oracle Database 12c, the speed with which histograms are collected has been `greatly improved`_.
 
Adaptive cursor sharing has a slight overhead though, as explained by the `Oracle Optimizer team`_: additional cursor memory, more soft and hard parses, and more child cursors.
The last one may cause cursor `cache contention`_.
 
The default setting for the ``CURSOR_SHARING``  parameter is ``'EXACT'``.
You can also set it to ``'FORCE'`` (`11g and 12c`_) or ``'SIMILAR'`` (`11g`_).
These settings are, however, generally recommended only as a temporary measure.
`Oracle's own recommendation`_ boils down to the following: ``'FORCE'`` is only used by lazy developers who cannot be bothered to use bind variables.

.. note::
   Oracle *never* replaces literals in the ``ORDER BY`` clause because the ordinal notation affects the execution plan: cursors with different column numbers in the ``ORDER BY`` cannot be shared.
 
Adaptive execution plans were introduced in 12c.
When we talked about execution plans we already mentioned the :ref:`mechanics <sql-adaptive>`: they allow execution plans to be modified on the fly.
On a development or test system adaptive cursor sharing and adaptive execution plans may mask underlying problems that need to be investigated and solved before the code hits production, which is why they should be switched off.
There are even some people who believe that these features have no place in a production system either because once you have determined the optimal execution plan, it should not be changed, lest you run into unexpected surprises.
As such, untested execution plans should never be released into the wild, according to `Connor McDonald and Tony Hasler`_.

.. _`expensive to compute, unstable when sampled, and the statistics have to be collected at the right time`: http://allthingsoracle.com/histograms-part-1-why
.. _`greatly improved`: http://jonathanlewis.wordpress.com/2013/07/14/12c-histograms
.. _`tanked at least once`: http://www.toadworld.com/platforms/oracle/b/weblog/archive/2014/03/04/oracle-12c-bind-variable-tips.aspx
.. _`Oracle Optimizer team`: http://blogs.oracle.com/optimizer/entry/how_do_i_force_a
.. _`cache contention`: http://dba-oracle.com/t_oracle_library_cache_contention_tips.htm
.. _`Oracle's own recommendation`: http://docs.oracle.com/database/121/TGSQL/tgsql_cursor.htm#TGSQL94750
.. _`Connor McDonald and Tony Hasler`: http://www.apress.com/9781430259770
.. _`11g and 12c`: http://docs.oracle.com/database/121/REFRN/refrn10025.htm#REFRN10025
.. _`11g`: http://docs.oracle.com/cd/B28359_01/server.111/b28320/initparams041.htm#REFRN10025