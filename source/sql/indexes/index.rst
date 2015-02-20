.. _sql-indexes:

*******
Indexes
******* 
Imagine you're feeling nostalgic one day and want to look for a definition in a printed dictionary;
you want to know whether 'crapulent' really is as funny as you think it is.
What makes the dictionary so practical when it comes to finding words and phrases?
Well, entries are listed from A to Z.
In other words, they are ordered.
All you need is the knowledge of the order of the letters in the alphabet.
That's all.
 
It would be very inefficient if you had to read the entire dictionary until you spotted the single item you're interested in.
In the worst case you could end up reading the entire 20 volumes of the Oxford English Dictionary, at which point you probably don't care about the assumed funniness of crapulence any longer.
 
The same goes for databases.
To fetch and retrieve rows in database tables it makes sense to have a simple and fast way to look them up.
That is what an index does.
It is similar to the index in old-fashioned books: the entries are listed in alphabetical order in the appendix, but the page number to which the entries refer does not generally have any structure.
That is, the physical order (when an entry is mentioned in the book) is independent of the logical order (when an entry is listed in the index, from A to Z).
In databases we typically do that with the aid of a `doubly linked list`_ on the index leaf nodes, so that each node refers to its predecessor and is successor; the leaf nodes are stored in a database block or page, which smallest available storage unit in the database.
This data structure makes it easy to run through the list in either direction.
 
The dictionary we mentioned earlier is an example of an index-organized table (IOT) in Oracle parlance;  Microsoft SQL Server calls these objects clustered indexes.
The entire table, or dictionary in our example, is ordered alphabetically.
As you can imagine, index-organized tables can be useful for read-only lookup tables.
For data sets that change frequently, the time needed to insert, update, and/or delete entries can be significant, so that IOTs are generally not recommended.
 
Where an index leaf node is stored is completely independent of its logical position in the index.
Consequently, a database requires a second data structure to sift quickly through the garbled blocks: a balanced search tree, which is also known as a `B-tree`_.
The branch nodes of a B-tree correspond to the largest values of the leaf nodes.
 
When a database does an `index lookup`_, this is what happens:
 
#. The B-tree is traversed from the root node to the branch (or header) nodes to find the pointers to relevant leaf node(s);

#. The leaf node chain is followed to obtain pointers to relevant source rows;

#. The data is retrieved from the table.
 
The first step, the tree traversal, has an upper bound, the index depth.
All that is stored in the branch nodes are pointers to the leaf blocks and the index values stored in the leaf blocks.
Databases can therefore support hundreds of leaves per branch node, making the B-tree traversal very efficient; the index depth is typically not larger than 5.
Steps 2 and 3 may require the database to access many blocks.
These steps can therefore take a considerable amount of time.
 
Oh, and in case you are still wondering: crapulent isn't that `funny at all`_.

.. _`doubly linked list`: http://en.wikipedia.org/wiki/Doubly_linked_list
.. _`index lookup`: http://www.orafaq.com/node/1403
.. _`B-tree`: http://use-the-index-luke.com/sql/anatomy/the-tree
.. _`funny at all`: http://www.oxforddictionaries.com/definition/english/crapulent



.. toctree::
   :hidden:

   Developer or Administrator <dev-or-admin>
   Access Paths and Indexes <access-paths>
   Statistics <statistics>
   Predicates: Equality before Inequality <predicates-equality-before-inequality>
   Predicates: LHS vs RHS <predicates-lhs-vs-rhs>
   Function-Based Indexes and NULLs <function-based-indexes>
   Predicates: WHERE clause <where-clause>
   Full Table Scans <full-table-scans>
   Top-N Queries and Paginations <top-n-pagination>
   Index-Organized Tables <index-organized-tables>
   Bitmap Indexes <bitmap-indexes>