.. _model-compression-performance:

Performance Considerations
==========================
Although size reduction is the main (and only) reason to switch on compression, it makes sense to look at the various option and how they affect the load times.


Size Reduction
--------------
Below is table with the size reduction compared to a simple uncompressed table. 
These numbers have been compiled from results published by `Uwe Hesse`_ (A), `Yannick Jaquier`_ (B), `Arup Nanda`_ (C), and `Talip Hakan Öztürk`_ (D). 
The values marked with an asterisk have not been included in the mean size reduction.

+------------------+--------------+------------+------------+------------+------------+
| Option           | **Size (%)** | Size A (%) | Size B (%) | Size C (%) | Size D (%) | 
+==================+==============+============+============+============+============+
| ``BASIC``        | **48**       | N/A        | 65.63      | 47.93      | 29.41      |
+------------------+--------------+------------+------------+------------+------------+
| ``OLTP``         | **59**       | N/A        | 78.68      | 56.26      | 41.17      |
+------------------+--------------+------------+------------+------------+------------+
| ``QUERY LOW``    | **19**       | 14.70      | 22.47      | 19.79      | N/A        |
+------------------+--------------+------------+------------+------------+------------+
| ``QUERY HIGH``   | **16**       | 8.82       | 13.44      | 0.52       | 41.17      |
+------------------+--------------+------------+------------+------------+------------+
| ``ARCHIVE LOW``  | **10**       | 6.62       | 12.81      | 0.52*      | N/A        |
+------------------+--------------+------------+------------+------------+------------+
| ``ARCHIVE HIGH`` | **8**        | 4.41       | 11.13      | 0.33*      | 29.41*     |
+------------------+--------------+------------+------------+------------+------------+

The numbers shown depend heavily on the data, so always be sure to try the different options with your own data! 
The percentages should *only* be read as an indication of the potential storage savings, not the gospel of Saint Oracle. 
You have been warned.

It's clear that the least compression is offered by ``OLTP``, which is not surprising. 
Similarly, we see that in most cases the hybrid column compression shaves off most disk space.

CPU Overhead
------------
What about data modification operations?

Talip Hakan Öztürk and Uwe Hesse have done extensive tests with the various options. 
Again, below are the summarized insert times compared to the ``BASIC`` option (not shown).

+------------------+---------------------+------------+------------+
| Option           | **Insert time (%)** | Time A (%) | Time D (%) |
+==================+=====================+============+============+
| ``OLTP``         | **316**             | 289.4      | 343.1      |
+------------------+---------------------+------------+------------+
| ``QUERY LOW``    | **69**              | 70.35      | 67.73      |
+------------------+---------------------+------------+------------+
| ``QUERY HIGH``   | **117**             | 147.9      | 85.40      |
+------------------+---------------------+------------+------------+
| ``ARCHIVE LOW``  | **145**             | 201.2      | 88.29      |
+------------------+---------------------+------------+------------+
| ``ARCHIVE HIGH`` | **710**             | 783.5      | 636.0      |
+------------------+---------------------+------------+------------+

The insert for the ``OLTP`` option is a conventional insert because in an OLTP environment a direct-path insert does not make sense. 
For the other compression methods the inserts are direct-path inserts. 
Note that conventional inserts generate a lot more undo and redo, which in itself is a cause for concern, especially with regard to performance.

Although there is quite a bit of variation in the insert times compared to the basic compression option, it seems that ``QUERY LOW`` achieves both a substantial size and time reduction. 
The most variation is visible for the hybrid columnar compression methods.

Yannick Jaquier reports that CPU time increases by 6.5 percentage points for the insert into a table with either ``ARCHIVE HIGH`` or ``QUERY HIGH`` compression as compared to the uncompressed case. 
However, the number of consistent gets for a full table scan are 88% and 90%, and the physical reads are 30% and 39% of that of an uncompressed table, respectively. 

Uwe Hesse has similar numbers. 
A full table scan takes 86.5% for ``BASIC``, 87.0% for ``OLTP``, 41.5% for ``QUERY LOW``, 61.3% for ``QUERY HIGH``, 54.6% for ``ARCHIVE LOW``, and 141.5% for ``ARCHIVE HIGH`` of the baseline's time, which is for an uncompressed table.

So, it's true that decompression does not affect read operations.
With the exception of ``ARCHIVE HIGH``  most compression options do not affect query and insert performance too badly.
It is possible (and likely) that the reduced I/O makes queries faster. 
Nice!

.. _`Uwe Hesse`: http://uhesse.com/2011/01/21/exadata-part-iii-compression/
.. _`Yannick Jaquier`: http://blog.yannickjaquier.com/oracle/data-compression-with-oracle-11gr2.html
.. _`Arup Nanda`: http://www.oracle.com/au/products/database/11g-compression-198295.html
.. _`Talip Hakan Öztürk`: http://taliphakanozturken.wordpress.com/tag/compress-for-archive-high