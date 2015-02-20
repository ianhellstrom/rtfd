.. _sql-hints-types:
 
Types of Hints
==============
Oracle has kindly provided `an alphabetical list`_ of all *documented* hints.
There are also a bunch of undocumented ones, and examples of their use can be found scattered all over the internet and in the multitude of books on Oracle performance tweaking.
Undocumented hints are not more dangerous than their documented equivalents; Oracle simply has not gotten round to documenting them yet.
 
Oracle classifies hints based on their function:
 
* Optimization goals and approaches;
* Access path hints;
* In-memory column store hints;
* Join order hints;
* Join operation hints;
* Parallel execution hints;
* Online application upgrade hints;
* Query tranformation hints;
* XML hints;
* Other hints.
 
In `Oracle Database 12c Performance Tuning Recipes`_, the authors provide two additional types of hints:
 
* Data warehousing hints;
* Optimizer hints.
 
The data warehousing hints are actually included in Oracle's query transformation hints.
 
Access path and query transformation hints are by far the largest two categories, save for the miscellaneous group.
 
Although interesting in their own way we shall not discuss in-memory column store hints, online application upgrade hints, and XML hints.
We shall now go through the remaining categories and discuss the most important hints for each category, so you too can supercharge your SQL statements.
There are many more hints than we describe here, and you are invited to check the official documentation for more hints and details.

.. _`an alphabetical list`: http://docs.oracle.com/database/121/SQLRF/sql_elements006.htm#SQLRF51108
.. _`Oracle Database 12c Performance Tuning Recipes`: http://www.apress.com/9781430261872



.. toctree::
   :hidden:

   Optimization Goals <goals>
   Optimizer Hints <optimizer>
   Access Path Hints <access-path>
   Join Order Hints <join-order>
   Join Operation Hints <join-operation>
   Parallel Execution Hints <parallel>
   Miscellaneous Hints <miscellaneous>