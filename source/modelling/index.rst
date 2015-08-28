.. _model-intro:

##############
Data Modelling
##############

`Data modelling`_ is typically the responsibility of database architects and to some degree DBAs.
The architects design the overall, high-level logical model and the administrators may give their opinion on the physical model, as they are probably the most knowledgeable when it comes to the internals of the technology to be deployed.

You may thus be wondering why data modelling pops up as a separate topic, as these pages clearly focus on optimization techniques for database *developers*, not architects or administrators.

The thing is, althrough some organizations are large enough to separate responsibilities as outlined, many software teams do not have the sheer size to split roles among members.
In some teams, especially DevOps, the various members may even need to assume various roles as required.
Moreover, there are a few topics regarding data modelling that are also of interest to developers, especially when it comes to performance.

One such topic is :ref:`partitioning <model-partition>`.
It is intimately tied to indexing, which is definitely a task for developers, as we have :ref:`already talked about <sql-indexes>`.

Another topic is data compression.
Anyone who has ever needed to send a large file via email knows that zipping or compressing it can make a huge difference. 
The same is of course true of the data stored in an Oracle Database, or any database for that matter.
Fortunately, Oracle offers several options for data compression for both data and indexes. 
We shall look at compression techniques because it is important you choose the right one for the task. 
After all, there is a performance trade-off: because the data gobbles up less space on the hard drive, the CPU has to do more work when adding or modifying data because it needs to compress and decompress the data on the fly. 

Anyway, let's quit yappin' and get right to business!

.. _`Data modelling`: http://www.agiledata.org/essays/dataModeling101.html

.. toctree::
   :hidden:

   partitioning/index
   compression/index