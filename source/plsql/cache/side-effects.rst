.. _plsql-cache-side-effects:
 
Side Effects
============
Before we talk about caching it is important to mention one important related topic: side effects.
 
When a subprogram (i.e. named PL/SQL block) changes only its own local variables, everything is just peachy.
However, when a subprogram modifies variables outside its scope we speak of side effects.
What this means for Oracle databases is that subprograms that are free of side effects cannot change global variables, public variables in packages, database tables, the database itself, external states, or their own ``OUT`` or ``IN OUT`` parameters.
 
What the heck is an external state?
Well, you can call ``DBMS_OUTPUT`` to send messages to a user's client or send an email with ``UTL_MAIL``.
That's the type of thing that is meant by an external state.
 
In `some programming circles`_, side effects are seen as evil as they do not confirm to the model of `pure functions` [#purefunc]_.
In essence, they consider side effects in medicine and computer science to be cut from the same cloth: they do something that is not intended and are generally seen to cause harm.
 
From a database developer's perspective that view is a bit too restrictive.
Changing table data is pretty much what a database is for, and you'd rather modify data through a PL/SQL API of your own making than type in the necessary IUD statements all day long.
 
Why care about `side effects`_?
Side effects can prevent parallelization of queries, yield order-dependent results, or require package states to be maintained across user sessions.
When it comes to caching, they can throw a spanner in the works.
It is thus good to know about side effects: the fewer side effects, the better your chances at using built-in caching mechanisms to supercharge your PL/SQL code!
 
.. _`some programming circles`: http://en.wikipedia.org/wiki/Functional_programming
.. _`pure functions`: http://en.wikipedia.org/wiki/Pure_function
.. _`side effects`: http://docs.oracle.com/database/121/LNPLS/subprograms.htm#LNPLS00814
 
.. rubric:: Notes
 
.. [#purefunc] A pure function is both idempotent and free of side effects. Note that idempotence enjoys different definitions in mathematics and functional programming versus and the rest of computer science. In mathematics, it means that for a function :math:`f \colon X \to Y` and :math:`x \in X`, :math:`f\left(f(x)\right) = f(x)`. In other words, we can *apply* the function as many times as we like but the result will be the same every time. The identity map :math:`x\mapsto 1` is a perfect example of an idempotent function in mathematics. In computer science, an idempotent function is one that can be *executed* many times without affecting the result.