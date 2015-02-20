.. _plsql-loops-collections:
 
Collections
===========
PL/SQL has three *homogeneous* one-dimensional collection types: associative arrays (PL/SQL or index-by tables), nested tables, and variable-size or varying arrays (varrays).
Homogeneous refers to the fact that the data elements in a collection all have the same data type.
Collections may be nested to simulate multi-dimensional data structures, but these are currently not supported with the traditional syntax you may be familiar with from other programming languages, such as C# or Java.
 
Summarized in the table below are the distinguishing features of each of these three collection types, where we have omitted ``[ CREATE [ OR REPLACE ] ] TYPE type_identifier IS ...`` from the declarations:
 
+---------------------+-------------------------------------------------+------------------+--------+-------+------------+----------------+------------+----------+---------------+
| Collection          | Type declaration                                | Index            | Sparse | Dense | Persistent | Initialization | ``EXTEND`` | ``TRIM`` |  ``MULTISET`` |
+=====================+=================================================+==================+========+=======+============+================+============+==========+===============+
| Nested table        | ``TABLE OF data_type``                          | positive integer | Yes*   | Yes   | Yes        | Yes            | Yes        | Yes      | Yes           |
+---------------------+-------------------------------------------------+------------------+--------+-------+------------+----------------+------------+----------+---------------+
| Associative array   | ``TABLE OF data_type INDEX BY ix_data_type``    | ``ix_data_type`` | Yes    | Yes   | No         | No             | No         | No       | No            |
+---------------------+-------------------------------------------------+------------------+--------+-------+------------+----------------+------------+----------+---------------+
| Variable-size array | ``VARRAY( max_num_elems ) OF data_type``        | positive integer | No     | Yes   | Yes        | Yes            | Yes        | Yes      | No            |
+---------------------+-------------------------------------------------+------------------+--------+-------+------------+----------------+------------+----------+---------------+
 
Here, ``ix_data_type`` can be either a ``BINARY_INTEGER``, any of its subtypes, or a ``VARCHAR2``.
Associative arrays are thus the only collection type that can handle *negative* and *non-integer* index values.

.. note::
   When it comes to performance, the difference between integers and small strings (i.e. fewer than 100 characters) as indexes is minimal.
   For large strings the overhead of hashing can be quite significant, as demonstrated by `Steven Feuerstein and Bill Pribyl`_.
 
We have added an asterisk to the 'Yes' in the column 'Sparse' for nested tables because technically they can be sparse, although in practice they are often dense;
they only become sparse when elements in the middle are deleted after they have been inserted.
The only collection type that can be used in PL/SQL blocks but neither in SQL statements nor as the data type of database columns is an associative array.
Although both nested tables and varrays can be stored in database columns, they are stored differently.
Nested table columns are stored in a separate table and are intended for 'large' collections, whereas varray columns are stored in the same table and thought to be best at handling 'small' arrays.
 
Nested tables and variable-size arrays require initialization with the default constructor function (i.e. with the same identifier as the type), with or without arguments.
All collections support the ``DELETE`` method to remove all or specific elements; the latter only applies to nested tables and associative arrays though.
You cannot ``DELETE`` non-leading or non-trailing elements from a varray, as that would make it sparse.
 
As you can see, the ``TRIM`` method is only available to nested tables and varrays; it can only be used to remove elements from the back of the collection.
Notwithstanding this restriction, associative arrays are by far the most common PL/SQL collection type, followed by nested tables.
Variable-size arrays are fairly rare because they require the developer to know in advance the maximum number of elements.
 
.. note::
   Oracle does not recommend using ``DELETE`` and ``TRIM`` on the same collection, as the results may be `counter-intuitive`_: ``DELETE`` removes an element but retains its placeholder, whereas ``TRIM`` removes the placeholder too.
   Running ``TRIM`` on a previously deleted element causes a deleted element to be deleted.
   Again.
 
The built-in ``DBMS_SQL`` package contains a couple of `collection shortcuts`_, for instance: ``NUMBER_TABLE`` and ``VARCHAR2_TABLE``.
These are nothing but associative arrays indexed by a ``BINARY_INTEGER`` based on the respective data types.
 
To iterate through a collection you have two options:
 
* A numeric ``FOR`` loop, which is appropriate for *dense* collections when the entire collection needs to be scanned: ``FOR ix IN l_coll.FIRST .. l_coll.LAST LOOP ...``.
* A ``WHILE`` loop, which is appropriate for sparse collections or when there is a termination condition based on the collection's elements: ``WHILE ( l_coll IS NOT NULL ) LOOP ...`` .
 
When using the ``FORALL`` statement on a sparse collection, the ``INDICES OF`` or ``VALUES OF`` option of the `bounds clause`_ may prove equally useful.
 
Now that we have covered the basics of collections, let's go back to the performance benefits of bulk processing.

.. _`Steven Feuerstein and Bill Pribyl`: http://shop.oreilly.com/product/0636920024859.do
.. _`counter-intuitive`: http://docs.oracle.com/database/121/LNPLS/composites.htm#CJAJAGII
.. _`collection shortcuts`: http://docs.oracle.com/database/121/ARPLS/d_sql.htm#CHDEEDCH
.. _`bounds clause`: http://docs.oracle.com/database/121/LNPLS/forall_statement.htm#LNPLS01321