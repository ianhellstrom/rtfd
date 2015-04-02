.. _plsql-cache:

*******
Caching
*******
 
A cache — pronounced like cash (i.e. money) — is the computer's equivalent of scrap paper to jot down information you may need to look up later.
The piece of paper is usually a fairly limited portion of memory in which the computer stores the information, which can be either computed values or even complete copies of data blocks, so it can find what it's after faster without having to redo the computation or look up the data from disk (again).
 
Oracle Database has three main methods for caching: one where the developer must do all the work and two that can be switched on and off with relative ease.
 
Of course, there is no such thing as a free lunch and that is true of caching as well.
You cannot cache everything 'just in case', because your memory is limited and the cache needs to be managed too.
In other words, there is an overhead associated with function caching in Oracle.
Nevertheless, enabling caching can lead to significant performance benefits that are well worth the memory sacrificed.
 
Interested to learn more about function caching in PL/SQL?
Then read on!



.. toctree::
   :hidden:

   Side Effects <side-effects>
   Alternatives <alternatives/index>
   The UDF Pragma <udf-pragma>
   The NOCOPY Directive <nocopy-directive>