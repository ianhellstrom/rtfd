.. _sql-basics:

**********
SQL Basics
**********
The histories of relational database management systems and SQL are inextricably linked.
Very few top-notch RDBMSs use a non-SQL programming language as the primary data manipulation language, although `a couple of alternatives have been spotted in the wild`_.
The implementations of SQL vary `from vendor to vendor`_, but most share roughly the same core feature set.

Relational database management systems, such as Oracle, are built on two pillars of mathematics: set theory and (first-order) predicate logic.
In a database live objects and these objects have certain relations to one another.
The objects have properties that are `quantifiable`_ and we can use these properties to compare objects.

All data is represented as *n*-ary relations.
Each relation consists of both a heading and a body.
A heading is a set of attributes, and a body of an *n*-ary relation is a set of *n*-tuples with no specific order of its elements.
A sets is an *unordered* collection of *unique* elements: the sets {a,b,c} and {b,c,a,c,c} are equivalent.
Whereas mathematicians generally use two-valued logic to reason about about data, databases use three-valued logic: true, false, and unknown (``NULL``).

Note that up to this point we have not talked about tables or columns at all.
So far we have been looking at the high-level conceptual database model, which consists only of entities (e.g. Location, Department, Product, Supplier, Customer, Sales, …) and relations.
The conceptual data model describes *what* we want to model.
When we move on to the logical data model, we need to add attributes and primary keys to our entities and relations.
We are still independent of the particulars of a database management system but we have to define *how* we want to model our entities and relations.
Common logical data models include the relational, network, hierarchical, flat, entity-relationship, and object-relational model.
Since Oracle is a relational database management system (RDBMS) we shall focus on that one here.

Once we know what we want to model and how we intend to model our high-level entities and relations, we must *specify* our logical data model, that is, we define our physical data model, which is highly dependent on the RDBMS we use: tables, views, columns, data types, constraints, indexes, procedures, roles, and so on.
Attributes are represented by columns, tuples by rows and relations by tables.
A nice, brief overview of the three levels of data models is available on `1Keydata`_.

With the risk of sounding pedantic, we wish to emphasize that tables are logical beasts: they have logical rows and columns.
Records and fields are their physical equivalents; fields are housed in the user interfaces of client applications, and records hang out in files and cursors.
We shall try and not confuse them but we don't want to make promises we can't keep.

Important to note is that rows can appear more than once in relational databases, so the idea that we can have only distinct elements in sets does not strictly apply; with the ``DISTINCT`` clause you can again obtain all unique elements but that is not really the point.
`Multisets`_ provide the appropriate generalization upon which RDBMSs are actually based, but even then SQL will deviate from the relational model.
For instance, columns can be anonymous.
Yes, we know: Oracle automatically assigns the expression of a column without an alias as the column name when outputting but that does not make it an actual attribute — try accessing it from outside a subquery or CTAS'ing into a new table without an ``ORA-00998`` error telling you to ``name this expression with a column alias``.
Anyway, we shall not dwell on any additional idiosyncrasies pertaining to the relational model.
In case you do crave for more details on the relational model though, we recommend the book *SQL and Relational Theory* by `Christopher J. Date`_.

.. _`a couple of alternatives have been spotted in the wild`: http://en.wikipedia.org/wiki/SQL#Alternatives
.. _`from vendor to vendor`: http://troels.arvin.dk/db/rdbms
.. _`quantifiable`: http://en.wikipedia.org/wiki/Quantification#Logic
.. _`1Keydata`: http://www.1keydata.com/datawarehousing/data-modeling-levels.html
.. _`Christopher J. Date`: http://www.amazon.com/SQL-Relational-Theory-Write-Accurate/dp/1449316409/
.. _`Multisets`: http://en.wikipedia.org/wiki/Multiset



.. toctree::
   :hidden:

   Style Guide <style-guide>
   Query Processing Order <query-processing-order>