.. _glossary:

########
Glossary
########

.. glossary::
   :sorted:
   
   SARGable
      Search ARGumentable.
      
   SGA 
      System Global Area: shared memory region.
      
   PGA
      Program Global Area: non-shared memory region that contains data and control information for a server process. 
      The PGA is created when the server process is started

   SQL Compiler
      The SQL compiler consists of the parser, the optimizer, and the row source generator. 
      It "compiles SQL statements into a shared cursor", where a cursor is simply a "handle or name for a private SQL area in the PGA". 
      A private SQL area "holds a parsed statement and other information, such as bind variable values, query execution state information, and query execution work areas".

   Access path
      An access path is a way in which a query retrieves rows from a set of rows returned by a step in an execution plan (row source). Below is a list of `possible access paths <http://docs.oracle.com/cd/E16655_01/server.121/e15858/tgsql_optop.htm#TGSQL228>`_:
  
      * full table scan
      * table access by ROWID
      * sample table scan
      * index unique scan
      * index range scan
      * index full scan
      * index fast full scan
      * index skip scan
      * index join scan
      * bitmap index single value
      * bitmap index range scan
      * bitmap merge
      * bitmap index range scan
      * cluster scan
      * hash scan