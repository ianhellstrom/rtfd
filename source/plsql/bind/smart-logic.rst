.. _plsql-bind-smart-logic:
 
Generic Static Statements
=========================
Developers sometimes write static SQL statements that are very generic in the sense that they accept different sets of parameters that are bound at runtime; some may be supplied, others may not.
The reason people go for such static code rather than a dynamic solution may be based on the misguided idea that dynamic SQL is slow.
This is what is referred to by `Markus Winand`_ as smart logic, probably because developers think they have created a nice, generic template.
 
An example might be as follows:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 11-13
 
    SELECT
      manufacturer
    , product
    , temperature
    , MIN(expiry_date) AS min_expiry_date
    , MAX(expiry_date) AS max_expiry_date
    , COUNT(*)             AS num_items
    FROM
      fridge
    WHERE
        ( product      = :prod OR :prod IS NULL )
    AND ( temperature  = :temp OR :temp IS NULL )
    AND ( manufacturer = :manu OR :manu IS NULL )
    GROUP BY
      manufacturer
    , product
    , temperature
    ;
 
In itself the query looks nice: it elegantly deals with many different cases thanks to the use of bind variables.
Swell!
 
The genericness of such a SQL statement is a fabulous from a coder's point of view.
From a performance perspective it is a tremendous weakness.
The reason is that the execution plan is the same in all instances.
 
Suppose for example that there is an index on ``manufacturer`` and ``product``.
 
**Question 1**: Will the index be used when the manufacturer of items in the fridge is given?
 
**Answer 1**: Unfortunately, no.
 
An index scan may be much more beneficial but Oracle does not know that.
There are 8 possible combinations of bind parameter values and there is no way that Oracle can capture the best plan in each of these cases with a single execution plan, especially since some predicates can benefit from the index whereas other cannot.
Hence, Oracle decides to do a full table scan and filter rows.
 
**Question 2**: Will the index be used when both manufacturer (``:manu``)  and product (``:prod``) are provided?
 
**Answer 2**: Nope.
 
An ``OR`` confounds index use.
Apart from that, we have already established that Oracle does not have a customized execution plan for this case, so that it reverts to a full table scan anyway.
 
**Question 3**: Will an index skip scan be used when only the product is supplied at runtime?
 
**Answer 3**: Again, no.
 
**Conclusion**: Oracle will *never* use an index for our query.
The issue with such generic pieces of static code is that the one-size-fits-all code leads to a one-plan-suits-(almost)-none situation.
The only cases where full table scans are *always* appropriate are when either only the temperature is known at runtime or all items from the fridge need to be returned.
Since ``temperature`` is not in our index it cannot benefit from an the index anyway.
For some special values of ``manufacturer`` and ``product`` a full table scan may also be the best solution, but it is doubtful that this applies to all possible cases.
 
The solution is to use dynamic SQL with bind variables and separate each case.
Alternatively, you can write a function that executes different SQL statements with bind variables based on the input of the function.
This entails that you will have a few more execution plans in the shared pool, but they are at least tuned to each instance rather than bad for all.
 
.. _`Markus Winand`: http://use-the-index-luke.com/sql/where-clause/obfuscation/smart-logic