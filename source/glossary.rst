.. _glossary:

########
Glossary
########

.. glossary::
   :sorted:

   access path
      An access path is a way in which a query retrieves rows from a set of rows returned by a step in an execution plan (row source). 
      Below is a list of `possible access paths <http://docs.oracle.com/cd/E16655_01/server.121/e15858/tgsql_optop.htm#TGSQL228>`_:
      
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

   anonymous block
      The basic unit of a PL/SQL source program is a `block <https://docs.oracle.com/database/121/LNPLS/block.htm#LNPLS01303>`_. 
      Each block must consist of at least one executable statement. 
      The declaration and exception-handling sections are optional. 
      Whenever a block is unnamed, it is known as an anonymous block. 
      Anonymous blocks are compiled each time they are loaded into memory:
     
      #. Syntax check: check PL/SQL syntax and generate a parse tree.
      #. Semantic check: check types and process the parse tree.
      #. Code generation.

   CGA
      The Call Global Area is a part of the :term:`PGA`. 
      It exists only for the duration of a call, that is, only as long as the process is running. 
      Anonymous blocks and data for top-level modules (i.e. functions and procedures) are in the CGA. 
      Package-level data is saved in the :term:`UGA`.

   instance
      A database instance, or instance for short, is a combination of :term:`SGA` and background processes. 
      An instance is associated with exactly one database. 
      Oracle allocates a memory area and starts background processes as soon as an instance is started.

   latch
      A latch is a mutual exclusion (mutex) device. 
      A process that requires access to a certain data structure that is latched (i.e. locked) tries to access the data structure intermittently (like a spinlock) until it obtains the information. 

      Compared to :term:`locks <lock>`, latches are lightweight `serialization <http://www.dba-oracle.com/t_lru_latches.htm>`_ devices. 
      Latches are commonly found in memory structures, such as the `SGA data structures <http://www.dba-oracle.com/t_difference_latch_lock.htm>`_.

      A `common gripe <http://samadhandba.wordpress.com/tag/shared-pool-latch/>`_ with contention for shared pool latches is hard parsing. 
      One way to reduce contention for latches in the shared pool is to eliminate literals by using bind variables as much as possible or set the ``CURSOR_SHARING`` parameter appropriately, although the former is recommended.
           
   lock
      A lock is a mutual exclusion (mutex) device. 
      A process enqueues until its request can be fulfilled in a first-come-first-serve (FCFS) manner. 
      Compared to :term:`latches <latch>`, locks are a `heavyweight synchronization mechanism <http://www.dba-oracle.com/t_difference_latch_lock.htm>`_. 
      Row-level locks are a primary example.

   PGA
      Program Global Area: non-shared memory region that contains data and control information for a server process. 
      The PGA is created when the server process is started.
      The PGA consists of the following areas that may or may not exist in every case:
     
      * SQL work area(s):
           
        * sort area
        * hash area  
        * bitmap merge area
           
      * session memory
      * private SQL area:
           
        * persistent area
        * runtime area
           
      A SQL work area is used for memory-intensive operations. 

      The private SQL area is the combination of the persistent and runtime areas. 
      The runtime area contains information about the state of the query execution, whereas the persistent area holds bind variable values. 
      A cursor is the name of a specific private SQL area, which is why they are sometimes used interchangeably.       
      
      The session memory is shared rather than private for shared server connections.
      Similarly, the persistent area is located in the :term:`SGA` for shared server connections.

   PL/SQL optimizer
      Since Oracle Database 10g the PL/SQL compiler can optimize PL/SQL code before it is translated to system code. 
      The optimizer setting ``PLSQL_OPTIMIZER_LEVEL`` --- 0, 1, 2 (default), or 3 --- determines the level of optimization. 
      The higher the value, the more time it takes to compile objects, although the difference is usually hardly noticeable and worth the extra time.

   processes
      There are two types of `processes <http://docs.oracle.com/database/121/CNCPT/process.htm#CNCPT1246>`_: Oracle processes and client processes. 
      A client process executes application or Oracle code. 
      Oracle processes come in three flavours: server, background processes, and slave processes. 
      A server process is one that communicates with a client processes and the database to fulfil a request:
     
      * Parse and execute SQL statements;
      * Execute PL/SQL code;
      * Read data blocks from data files into the buffer cache;
      * Return results.
     
      Server processes can be either dedicated or shared. 
      When the client connection is associated with one server process, we speak of a dedicated server connection. 

      In shared server connections, clients connect to a dispatcher process rather than directly to a server process. 
      The dispatcher receives requests and places them into the request queue in the large pool (see :term:`SGA`). 
      Requests are handled in a first-in-first-out (FIFO) manner. 
      Afterwards, the dispatcher places the results in the response queue.

      Background processes are automatically created when an instance starts. 
      They take care of for example maintenance and recovery (redo) tasks. 
      Mandatory background processes include:
       
      * PMON: process monitor process;
      * LREG: listener registration process;
      * SMON: system monitor process;
      * DBW: database writer process;
      * LGWR: log writer process;
      * CKPT: checkpoint process;
      * MMON/MMNL: manageability monitor process;
      * RECO: recoverer process.
       
      Slave processes are background processes that perform actions on behalf of other processes. 
      Parallel execution (PX) server processes are a classical example of slave processes.

   PVM
      The PL/SQL vitual machine (PVM) is a database component that executes a PL/SQL program's bytecode. 
      Inside the VM the bytecode is translated to machine code that is executed on the database.
      The intermediate bytecode (or MCode for machine code) is stored in the data dictionary and interpreted at runtime.
      
      `Native compilation <http://www.oracle.com/technetwork/database/features/plsql/ncomp-faq-087606.html>`_ is a different beast altogether.
      When using PL/SQL native compilation, the PL/SQL code is compiled into machine-native code that bypasses the interpretation at runtime.
      The translation of PL/SQL code into a shared C library requires a C compiler; these shared libraries are not portable.

   SARGable
      Search ARGumentable.
          
   SGA
      The System Global Area contains data and control information. 
      It consists of the shared pool, the database buffer cache, the redo log buffer, the Java pool, the streams pool, the in-memory column store, the fixed SGA, and the optional large pool. 
      It is shared by all server and background processes.  
      The so-called large pool is an `optional area <http://docs.oracle.com/database/121/CNCPT/memory.htm#BGBGHJAA>`_ in the SGA that is intended for memory allocations that are larger than is appropriate for the shared pool. 
      This is used to avoid memory fragmentation.

   shared pool
      The shared pool is an area in the :term:`SGA` that consists of the library cache, the data dictionary cache, the server result cache, and the reserved pool. 
      The library cache holds the shared SQL area, and, in the case of a shared server connection, also the private SQL areas.

   SQL compiler
      The SQL compiler consists of the parser, the optimizer, and the row source generator. 
      It "compiles SQL statements into a shared cursor", where a cursor is simply a "handle or name for a private SQL area in the :term:`PGA`". 
      A private SQL area "holds a parsed statement and other information, such as bind variable values, query execution state information, and query execution work areas".

   UGA
      The User Global Area is the memory associated per user session. 
      Package-level data is stored in the UGA, which means that it grows linearly with each new user session. 
      When the state of a package is serially reusable (``PRAGMA SERIALLY_REUSABLE``), the package data is stored in the :term:`SGA` and persists for the life of the server call. 
      Non-reusable package states remain for the life of a session.
