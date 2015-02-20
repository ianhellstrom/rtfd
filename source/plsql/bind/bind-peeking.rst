.. _plsql-bind-peeking:
 
Bind Peeking
============
If bind variables are so grand, why not enable them by default, everywhere?
 
The problem lies in what is referred to as bind peeking.
When Oracle encounters a statement with bind variables for the very first time, it looks at the literals supplied, checks the histogram (if available), and then fixes the execution plan.
 
In itself this seems reasonable, right?
When data is highly skewed that may become an issue.
Let's go back to our ``fridge`` table and fill it in accordance with the appetite of many developers: beer, beer, and some more beer.
Because we have a few guests over tonight we also buy some white wine, salad, and avocados; please, don't ask why.
We have created a histogram too: 95% of the rows are related to beer, whereas the remaining 5% are split among the three newly acquired products.
 
When we send our friendly household robot to run to the kitchen to get beer, and the query contains a bind variable for product, a full table scan will be used.
The next time we send it to get white wine and it still uses a full table scan even though an index lookup would be much faster.
Why does it do a full table scan?
For the first execution our database robot peeked at the bind variable's value, and because its value was ``'Beer'``, the execution plan was fixed with a full table scan.
The second execution did not care about the fact that we wanted ``'White Wine'`` since the robot had already decided that the execution plan involved a full table scan.
 
The reverse situation is also possible.
An index scan is used based on the initial execution of the request for something other than beer, even though a full table scan is much more advantageous when we desire beer.
Thus, as stated by `Arup Nanda`_, "smart SQL coders will choose when to break the cardinal rule of using bind variables, employing literals instead."

.. _`Arup Nanda`: http://www.oracle.com/technetwork/articles/sql/11g-sqlplanmanagement-101938.html