.. _rationale:

***************
Why This Guide?
***************
While it's easy to write bad code in any programming language, SQL — and to some extent PL/SQL too — is particularly susceptible.
The reason is simple: because of its natural-looking syntax and the fact that a lot of technical 'stuff' goes on behind the scenes, some of which is not always obvious to all but seasoned developers and DBAs, people often forget that they are still dealing with a programming language and that SQL is not a silver bullet.
Just because it looks easy does not mean that it is.
We don't want to step on anyone's toes but frequently production SQL code (e.g. reports) is created not by developers but business users who often lack the know-how to create quality code. It's not necessarily their jobs to come up with efficient queries but it is not uncommon to hear their gripes afterwards, once they discover that a report takes 'too long' because the database is 'too slow'.
The fact of the matter is that they have run into one of many traps, such as non-SARGable predicates, bad (or no) use of indexes, unnecessarily complicated (nested) subqueries that not even their creator can understand after a short lunch break but somehow magically deliver the correct, or rather desired, results.
Other programming languages, especially the more common third-generation ones, do not have that problem: applications are mostly developed by professional developers who (should) know what they're doing.

There are many excellent references on the topic of SQL and PL/SQL optimization, most notably Oracle's own extensive `documentation`_, Tom Kyte's `Ask Tom`_ Q\&A website, entries by `Burleson Consulting`_,  Tim Hall's `Oracle Base`_ pages, Steven Feuerstein's `PL/SQL Obsession`_, `Oracle Developer`_, `books`_, and a wealth of blogs (e.g. by `Oracle ACEs`_).

'So why this guide?' we hear you ask.
Two reasons really:

#. It's a long and at times arduous journey to master Oracle databases, and it's one that never ends: Oracle continues to develop its flagship database product and it is doubtful that anyone knows everything about its internals.
   We hope that other developers will benefit from the experiences (and rare insights) chronicled here.
   These page are personal field notes too, so they evolve as we discover new avenues of investigation.
   The bulk of what is written here can of course be found elsewhere, and we have included references where appropriate or rather where we remembered the respective sources.
   Since it it is rarely straightforward to gather the information relevant to your particular use case, sift through it in a timely manner, and understand it well enough to use it, we hope to assist in that task.
   We have no intention or illusion of replacing the aforementioned resources or the coder's home base: `Stack Overflow`_.

#. A lot of the documentation on tuning queries is geared towards DBAs.
   Often a developer needs to work closely with the DBA to discover what the source of some performance issue is, and quite frequently the DBA can help: if the database is tuned improperly, the best query in the world can be excruciatingly slow.
   Most of the time, though, the developer can tweak his or her queries without calling in the cavalry.
   These pages describe what a SQL and/or PL/SQL developer can do without the DBA.
   Furthermore, code optimization is often seen as the last step in the development process, and it is sometimes only then looked at seriously when the performance is abominable.
   We believe that much can be done before it's (almost) too late.

.. note::
   We do not advocate premature optimization at all; code does not have to be as fast as technically possible, just as fast as necessary.
   Instead we want to stress that basic knowledge of the database engine and SQL optimizer can help you avoid common pitfalls.
   By doing so, you will not only learn to be more productive but also steer clear of many headaches and frowns from fellow developers along with their flippant references to RTFM, usually followed by a gesture to signal that you are to leave the office quickly.

.. _`documentation`: http://www.oracle.com/technetwork/documentation/index.html#database
.. _`Ask Tom`: http://asktom.oracle.com
.. _`Burleson Consulting`: http://www.dba-oracle.com
.. _`Oracle Base`: http://www.oracle-base.com
.. _`PL/SQL Obsession`: http://www.toadworld.com/platforms/oracle/w/wiki/8243.plsql-obsession.aspx
.. _`Oracle Developer`: http://www.oracle-developer.net/
.. _`books`: http://www.amazon.com/s/ref=nb_sb_noss?url=node%3D5&field-keywords=oracle+tuning+optimization&rh=n%3A283155%2Cn%3A5%2Ck%3Aoracle+tuning+optimization
.. _`Oracle ACEs`: http://apex.oracle.com/pls/apex/f?p=19297:3:
.. _`Stack Overflow`: http://stackoverflow.com/questions/tagged/oracle