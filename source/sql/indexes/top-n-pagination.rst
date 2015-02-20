.. _sql-indexes-pagination:
 
Top-N Queries and Pagination
============================
Top-N and pagination queries frequently pop up in applications: a user is only shown the top-N entries or allowed to flip back and forth between pages of data.
Prior to Oracle Database 12c there were a couple of `roundabout methods`_ available to do pagination: `offset, seek`_, `window or analytical functions`_.
 
The ``OFFSET/FETCH`` or `row-limiting clause`_ has greatly simplified life for developers:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 10,11
 
   SELECT
      manufacturer
    , product
    , temperature
    , expiry_date
   FROM
      fridge
   ORDER BY
      expiry_date
   OFFSET 5 ROWS
   FETCH NEXT 10 [ PERCENT ] ROWS ONLY  
   ;

An issue that is often overlooked when it comes to the row-limiting clause is explained on `Markus Winand's Use The Index, Luke`_ page.
We'll briefly cover the salient details, as it affects application and database performance.
Suppose your users flip through pages of data and are allowed to insert rows at any position.
The ``OFFSET`` clause can cause rows to show up twice: once on the previous page *before* the row was inserted and once on the current page *after* the row was inserted (on the previous page).
Furthermore, ``OFFSET`` is implemented in a way that data below the ``OFFSET`` line needs to be fetched and sorted anyway.
 
The solution to this conundrum is quite simple: keyset pagination: use the ``FETCH`` clause as before but replace the ``OFFSET`` clause with a ``WHERE`` clause that limits the result set to all rows whose key is before or after the identifier (key) of the row previously displayed.
Whether you have to take ``>`` or ``<`` depends on how you sort and what direction the pagination runs in of course.
An index on the columns in your ``WHERE`` clause, including the key, to aid the ``ORDER BY`` means that browsing back to previous pages does not slow your users down.
 
With that in mind we can rewrite our query:
 
.. code-block:: sql
   :linenos:
   :emphasize-lines: 9,12

   SELECT
      manufacturer
    , product
    , temperature
    , expiry_date
   FROM
      fridge
   WHERE
      expiry_date < last_expiry_date_of_previous_page
   ORDER BY
      expiry_date
   FETCH NEXT 10 [ PERCENT ] ROWS ONLY
   ;
 
Two major bummers of keyset pagination are that 1) you cannot jump to arbitrary pages because you need the values from the previous page and 2) no convenient bidirectional navigation is available because that would require you to reverse the ``ORDER BY`` and key comparison operator.

.. _`offset, seek`: http://use-the-index-luke.com/sql/partial-results/fetch-next-page
.. _`roundabout methods`: http://www.oracle-base.com/articles/12c/row-limiting-clause-for-top-n-queries-12cr1.php
.. _`window or analytical functions`: http://use-the-index-luke.com/sql/partial-results/window-functions
.. _`Markus Winand's Use The Index, Luke`: http://use-the-index-luke.com/no-offset
.. _`row-limiting clause`: http://docs.oracle.com/database/121/SQLRF/statements_10002.htm#SQLRF55631