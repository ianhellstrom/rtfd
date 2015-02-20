.. _sql-hints-when-not:
 
When Not To Use Hints
=====================
Even though Oracle -- ahem -- hints us to use hints as a last resort, Oracle whizz `Jonathan Lewis goes even further`_ and pretty much has one simple rule for hints: don't.
 
We do not take such a grim view of the world of hints but do wish to point out that there are actually very good reasons not to use hints at all.
 
A common use case for hints is when statistics are out of date.
In such cases hints can indeed be useful, but an approach where representative statistics are locked, such as the TSTATS approach, may be warranted and in fact more predictable.
Hints are basically static comments that fiddle around with the internals of the optimizer.
Dynamically generated hints (i.e. hints generated on-the-fly in dynamic SQL) are *extremely* rare, and come to think of it, we have never seen, heard of, or read about them anywhere.
Full stop. [#dynhints]_
 
Because hints are static they are in some ways the same as locked statistics.
The only difference is that once you have seen (in development) that different statistics are more appropriate and you need to do a refresh, you can release the statistics into production by copying the statistics from development to production.
Oracle will take care of the rest, and the optimizer can figure out the new best execution plan on its own, which will be the same as in development because the statistics are identical.
You really don't want to run into performance surprises in production, especially if they suck the life out of your statements.
 
With hints, you have to check the performance against both the development and the production instance(s), as the statistics may very well be different.
What you lose with hints is the predictability of locked statistics.
You also need to regularly verify that the hints still perform as initially intended.
And typically that is not something you want to do in a live system.
 
Moreover, when Oracle releases a new version of its flagship database, it usually comes packed with lots of improvements to the optimizer too.
If you have placed hints inside your code, the optimizer does what you tell it to do, and you are unlikely to benefit from any new features.
Some of these features may not always be helpful but in many cases they really are.
 
This brings us to another important point.
Hints should be revisited regularly and documented properly.
Without documentation no one except you during a period of a few hours or perhaps days, depending on nimbleness of your short-term memory and the amount of beer and pizza consumed in the time afterwards, will know why the hints were required at all, and why that particular combination of hints was chosen.
 
In summary, don't use hints when
 
* what the hint does is poorly understood, which is of course not limited to the (ab)use of hints;
* you have not looked at the root cause of bad SQL code and thus not yet tapped into the vast expertise and experience of your DBA in tuning the database;
* your statistics are out of date, and you can refresh the statistics more frequently or even fix the statistics to a representative state;
* you do not intend to check the correctness of the hints in your statements on a regular basis, which means that, when statistics change, the hint may be woefully inadequate;
* you have no intention of documenting the use of hints anyway.

.. _`Jonathan Lewis goes even further`: http://jonathanlewis.wordpress.com/2008/05/02/rules-for-hinting

.. rubric:: Notes

.. [#dynhints] Even though we have never observed dynamically generated hints in the wild we can still perform a Gedankenexperiment to see why they seem like an odd idea anyway. Suppose you want to provide cardinality estimates with the undocumented ``CARDINALITY`` hint based on the parameter values of a subprogram, for instance the parameter of a table function. You may think this is a great idea because you already know about skew in your data, and you want to provide estimates based on your experience. Fine. Unfortunately, you cannot bind the estimate itself, which means that Oracle requires a hard parse, as the hint is simply a literal. This is tantamount to hard-coding the hint and choosing the statement to run with branches of a conditional statement, which sort of defeats the purpose of generating the estimate dynamically. Creating several alternatives based on parameter values may, however, be useful and beneficial to the performance, especially in cases of severe data skew.


