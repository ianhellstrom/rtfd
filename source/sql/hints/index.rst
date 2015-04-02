.. _sql-hints:

*****
Hints
*****
According to the `Oxford Dictionary`_, a hint is "a slight or indirect indication or suggestion".
Oracle optimizer hints can be broadly categorized in two classes: 1) real hints, and 2) instructions.
The former, real hints, match the dictionary definition: they provide Oracle with pertinent information that it may not have when it executes a particular statement.
The information provided manually, for instance a cardinality estimate of an object, may aid in the search for an optimal execution plan.
The latter are really best called `instructions`_ or directives as they tell the optimizer what (not) to do.
 
Hints are actually comments in SQL statements that are read by the optimizer.
They are indicated by the plus sign in ``/*+ SOME_HINT */``.
Without the plus sign the comment would be just that: a comment.
And the optimizer does not care about the comments you add to increase the quality or legibility of your code.
With the plus sign, the optimizer uses it do determine the execution plan for the statement.
 
As always, the hint may be written in any case (UPPER, lower, or miXEd) but it must be valid for the optimizer to be able to do anything with it.
Whether you add a space after the '``+``' and/or before the closing '``*/``' is up to you; Oracle does not care either way.
 
It is customary to place the hint directly after the SQL verb (e.g. ``SELECT``) but it is not necessary to do so.
However, the hint must follow the plus sign for the optimizer to pick it up, so do not place any comments in front of the hint.
In fact, we recommend that you do not mix comments and hints anyway.
Several hints may be placed in the same 'comment'; they have to be separated by (at least) one space, and they may be on different lines.
 
Mind you, the syntax checker does *not* tell you whether a hint is written correctly and thus available to the optimizer!
You're on your own there.
If it's not valid, it's simply ignored.
 
We shall henceforth use upper-case hints, lower-case parameters that have to be provided by the developer, and separate all parameters by two spaces to make it easier for you to read the hints.
Some people prefer to use commas between parameter values but that is not strictly necessary, so we shall leave out the commas.

.. _`Oxford Dictionary`:  http://www.oxforddictionaries.com/definition/english/hint
.. _`instructions`: http://allthingsoracle.com/a-beginners-guide-to-optimizer-hints



.. toctree::
   :hidden:

   When To Use Hints <when-to-use>
   When Not To Use Hints <when-not-to-use>
   Named Query Blocks <named-query-blocks>
   Global Hints <global-hints>
   Types of Hints <types/index>
   Optimization Techniques <optimization>