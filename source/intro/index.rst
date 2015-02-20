.. _intro:

############
Introduction
############
SQL is a peculiar language.
It is one of only a handful of fourth-generation programming languages (4GL) in general use today, it seems deceptively simple, and more often than not you have many quite disparate options at your disposal to get the results you want, but only few of the alternatives perform well in production environments.
The simplicity of the language veils decades of research and development, and although the syntax feels (almost) immediately familiar, perhaps even natural, it is a language that you have to wrap your head around.
People coming from imperative languages often think in terms of consecutive instructions: relational databases operate on sets not single entities.
How the SQL optimizer decides to execute the query may not coincide with what you think it will (or ought to) do.

To the untrained eye a database developer can seem like a sorcerer, and it is often said that query tuning through interpreting execution plans is `more art than science`_.
This could not be further from the truth: a thorough understanding of the inner workings of any database is essential to squeeze out every last millisecond of performance.
The problem is: the information is scattered all over the place, so finding exactly what you need when you need it can be a daunting task.

.. _`more art than science`: http://www.google.de/search?q=sql+"execution+plan"+"more+art+than+science"



.. toctree::
   :hidden:

   Why This Guide? <rationale>
   System and User Requirements <prerequisites>
   Notes <notes>