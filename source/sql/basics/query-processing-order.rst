.. _sql-basics-proc-order:

Query Processing Order
======================
Important to understand before we discuss execution plans is how Oracle processes queries logically.
Let's look at the following query:

.. code-block:: sql
   :linenos:
   
   SELECT 
       f.product AS beer
     , p.product AS crisps
   FROM
     fridge f 
   CROSS JOIN
     pantry p 
   WHERE
     f.product         = 'Beer'
     AND f.temperature < 5 
     AND f.size        = '50 fl oz' 
     AND p.product     = 'Crisps'
     AND p.style       = 'Cream Cheese' 
     AND p.size        = '250g'
   ORDER BY
       crisps
     , beer
   ;

What does this query tell you other than that you're a tad peckish, extremely thirsty, and that the fridge and pantry seem to use different systems of measurement?

You may think that it reads the query in the way that we type it, but Oracle (and other RDBMSs too) does not read from top to bottom.
It more or less reads our queries upside down.
Not exactly, but we'll see what it does in a minute.
Oracle is a fancy machine that translates our SQL statements into something it can understand and execute.
In essence, it's a data robot that does exactly what we tell it to do.
Now, suppose you have purchased a robot to help you around the house, and its first and foremost task is to assist you in quenching your thirst and sating your appetite.
How would you tell it go fetch a beer and a packet of crisps?

Well, you'd probably tell it to go to the fridge, look for beer, grab a bottle (50 fl oz) with a temperature below 5 degrees Celsius, then go to the pantry and look for a 250g packet of cream cheese crisps. Once it's done, it should come back to you and place the items in front of you, sorted in the way you asked it to do.
That's right, you first tell it to go to the place where the fridge and the pantry are located (probably the kitchen: ``FROM``), then to look for everything that matches your criteria (``WHERE``), and finally to return the items (``SELECT``) sorted in the order you specified (``ORDER BY``).

That's pretty much what Oracle does too. 
The order in which clauses are logically processed by Oracle is as follows: ``FROM -> CONNECT BY -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY``.

Of course, your query does not have to have every clause, and some cannot even be used with/without others (e.g. ``HAVING`` can only be used when you use ``GROUP BY``).

.. _fig-proc-order:

.. figure:: ../../images/query-proc-order.*
   :scale: 60%
   :alt: query processing order
   
   Oracle's query processing order, including optional clauses.

The processing order is also the reason why the previous query worked like a charm and the following will result in an error:

.. code-block:: sql
   :linenos:
   
   SELECT
       product          AS item
     , MIN(temperature) AS min_temperature
     , COUNT(*)         AS num_products
   FROM
     fridge
   GROUP BY
     item
   ;         

When Oracle processes the ``GROUP BY`` clause the alias ``item`` is not yet known, so you are greeted by an ``ORA-00904: invalid identifier`` error.
