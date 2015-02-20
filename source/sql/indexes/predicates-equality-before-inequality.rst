.. _sql-indexes-predicates:
     
Predicates: Equality before Inequality
======================================
An index can be beneficial to your queries' performance when there is some sort of filtering that can be handled efficiently by the index.
The performance is intimately related to the ``WHERE`` clause and the existence (or absence) of indexes on columns specified in the ``WHERE`` clause.
As such, the ``INSERT`` statement is the only one of the unholy insert-update-delete (:term:`IUD <IUD statements>`) trinity that can never benefit from an index: it has no ``WHERE`` clause.
With an ``UPDATE`` or ``DELETE`` you typically have predicates, so they can benefit from fast lookups, even though the maintenance of the index negatively affects the performance; it is a trade-off that has to be evaluated carefully by the developer.
In data warehouse environments it is not uncommon to drop all indexes temporarily and re-create them once the data loaders have completed their business.
Alternatively, you can make your index unusable (i.e. ``ALTER INDEX index_name UNUSABLE``) and once you have imported the data, rebuild it: ``ALTER INDEX index_name REBUILD``.
Only for function-based indexes you can ``DISABLE``  and ``ENABLE`` the index.
 
Predicates show up in execution plans as access, index filter, or table-level filter predicates.
An access predicate corresponds to the start and stop conditions of the leaf node traversal.
During the leaf node traversal index filters can be applied.
In case you filter on a column that is not in the index, the filter predicate is evaluated on the level of the table.
 
It is important that you understand the differences in predicates, because it is critical to the index range scan access method.
Access predicates limit the range of the index to be scanned, whereas index filters are applied to the results of the scanned range.
This is typically the performance bottleneck for index range scans;
if the requested rows are all stored in the same block, then it may not be a problem because Oracle can use a single read operation to fetch the data.
However, you want to keep the scanned index range as small as possible, because the leaf node traversal can take a while.
 
Because index maintenance affects the performance of insert, update, and delete (IUD) statements, it is important that you create enough indexes but no more than needed.
The fewer indexes a table has, the better its IUD performance; the fewer indexes the execution of a query uses, the better its performance.
The question thus is how to optimally define indexes, so that the performance of queries is acceptable while at the same time the performance of IUD statements is not abominable.
       
Primary key and unique constraints automatically generate database indexes.
Sometimes these constraints are sufficient but sometimes you need more.
 
If you only have one column that you ever use to look for data, then that column is your index.
Suppose your application requires an ISBN to return the title, the first and last name of the main author, purely as information and nothing else.
It can then make sense to include these three columns in the index too, not because you filter on them — because we just said you don't — but to make sure that Oracle can avoid a trip to the table: Oracle can simply read the index and retrieve the information it needs to provide from it.
Such retrieval logic is generally coded in PL/SQL functions that live inside the database.
We shall talk more about functions when we talk about PL/SQL, but we want to mention two advantages of this approach to whet your appetite: when using bind variables, the execution plan is stored irrespective of the ISBN provided, so Oracle does not need to ask the optimizer again and again, and you can take advantage of the result cache, which means that information on popular books is cached and available without having to check the index or fetch data from the table.
 
Anyway, when you include multiple columns in an index, that is you create a compound index, the order of the columns is very important.
The difference between a column being used as a filter rather than access predicate can be significant.
Because of that, it is recommended to index for `equality first`_ and then for ranges, typically ``DATE``-like columns, so that the extent of the leaf node traversal can be kept to a minimum.
 
Let's take another look at our :ref:`friendly household robot <sql-basics-proc-order>`.
We wanted beer chilled below five degrees.
The time we have to wait for the robot to fetch the items from the fridge depends on how fast our electronic companion can locate the items requested.
An index is of course beneficial but the question remains what we want to index first: temperature or product.
Because we rarely ever need an item at an exact temperature, that is we want items in a particular temperature range, it makes sense to index on product first and then on temperature.

'What's the difference?' you ask?
Well, let's find out.

.. _fig-index-range-eq:

.. figure:: ../../images/index-nodes-range-eq.*
   :scale: 60%
   :alt: Index branch and leaf nodes for range (temperature) and then equality (product)
 
When the index is on temperature first and then product, the robot checks one of the branch nodes and sees that there are *at least* two leaf nodes with the correct temperature; the temperature is the access predicate.
It follows the pointers to these leaf nodes and uses the product column as a filter predicate.
Because one of the leaf nodes' maximum value for temperature is 4, we have to follow the leaf node chain because the next leaf node may contain more products with a temperature below five degrees.
And yes, there is one more item; it's not what we were looking for but our robot could not know that when browsing the index.

.. _fig-index-eq-range:

.. figure:: ../../images/index-nodes-eq-range.*
   :scale: 60%
   :alt: Index branch and leaf nodes for equality (product) and then range (temperature)
 
Assume the product is index first and the temperature second, as recommended.
What's the advantage?
Well, the index tells us exactly in which leaf nodes to look for beer: it's in the ones before cream because ``'apple' < 'beer' < 'cream'``.
We still have the case that we have to follow the leaf node chain because ``'beer'`` happens to be the last product in one of the index leaf nodes.
 
Note that the advice about equality-range indexes is not the same as saying that the most selective index in our case should go first.
Suppose, for the sake of argument, that each product in our table has a different temperature.
Depending on the temperature of the warmest item, we could have to scan the entire table, if all products have been in the fridge at least a few hours, so that they are all below five degrees, or we could have no items to consider at all because someone forgot to close the fridge door and everything is almost at room temperature.
Sure, in the latter case an index on the temperature first would yield the best performance.
But: in the former case the performance could potentially be horrible, especially if the number of products in the fridge were to increase significantly.
 
Another scenario to ponder about: all products in the fridge are unique.
For the product-temperature index we would look up the product and then verify whether its temperature is in the specified range.
Simple. Quick.
For the temperature-product index, however, we could potentially have to scan everything and then filter out the items that are not beer.
 
Yet another scenario: all products in the fridge are beer — man, you're thirsty!
The product-temperature index requires us to do a full fridge scan and take only the bottles below five degrees.
The temperature-product index is obviously more efficient because it can use the temperature as an access predicate; the filter predicate on ``'beer'`` is pretty much useless, as is the index.
 
As you can see, the performance of an equality-range index is more consistent and thus more production-suitable than the range-equality index.
There are of course cases when the range-equality index is more appropriate: skewed data, where you *know* it is more advantageous, but you have to be absolutely sure the skewness stays the same.
For most databases that is a bit too iffy to be useful advice.
 
Another reason why the equality-range index is a good rule of thumb is that whatever is searched for with an equality predicate is something that is pretty much standard to all your queries: you primarily want certain stuff from the fridge, where the temperature is only secondary.
"I'm starving for some 7-degrees' produce," is not something you often hear people say when they're thirsty but have no cool liquids available; you might want to gobble up some cool broccoli instead but I doubt it.
 
If our fridge table is equipped with expiry dates, that column would also be included as a second or third column.
We're typically interested in items that have not yet expired (``expiry_date <= SYSDATE``), or, if we want to have the robot clean up the fridge, all items that have already expired.
Whether the temperature or expiry date should go first *after* the product depends a bit on the situation: do you search more frequently for the expiry date or the temperature of items in the fridge?
 
Anyway, when you need an index on additional columns, add these to the index you already have or redefine it.
An extra index may not provide you with the benefits you expect: the optimizer has to combine two indexes when executing your queries, and the database has to maintain two indexes.
The fewer indexes the optimizer has to use, the better the performance of your queries.
More than 5 indexes is usually not recommended, but the exact number may well depend on the specifics of your environment.
Nevertheless, if you are really looking at five or more indexes for a particular table, you have to think about why you need so many separate indexes, and document your reasons carefully.
 
With regard to SQL statements, always be as specific as possible.
Suppose you go to the trouble of adding manufacturers of products in your fridge, you create a compound manufacturer-product index, and let the legged circuit board look for some ``'Coke Zero'`` by ``'The Coca-Cola Company'``.
Sure, ``'Coke Zero'`` is only made by one company, but today you're too tired, so you simply write ``WHERE product = 'Coke Zero'``.
If you're lucky, the robot decides to do a skip scan on the leading edge of the index; if you are less fortunate, and your fortune depends mainly on the histogram of the leading index column (i.e. the manufacturer column) , your robot may decide on a full fridge scan.
Oracle does not know about correlations in your data, so if you want Oracle to come back with your rows as quickly as possible, provide all the details possible that aid it in its search for your data.
If at all possible, always include your leading index column in all your queries' predicates.
It is advice given to mathematicians and, likewise, applies to (database) developers: do not assume anything unless stated otherwise.

.. _`equality first`: http://use-the-index-luke.com/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates