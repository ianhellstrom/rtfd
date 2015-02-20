.. _plsql-bind:

**************
Bind Variables
**************
Oracle Database developers drool when they hear the phrase "bind variables".
Well, not literally, but it is often cited as the key to solid application performance.
 
When SQL statements are littered with literals, the shared pool is likely to be cluttered with many similar statements.
When you execute the same statement over and over again but each time with different literals in the ``WHERE`` clause, you cause Oracle to parse the statement every time it is executed.
In many cases, particularly when the data isn't horribly skewed, the execution plan is the same.
Such a hard parse is a waste of valuable resources: the parsing itself costs CPU time and the various cursors bog up the shared pool.
 
"But what can we do?"
 
Use bind variables!
 
Straight from the `horse's mouth`_: "[a] bind variable is a placeholder in a SQL statement that must be replaced with a valid value or value address for the statement to execute successfully. By using bind variables, you can write a SQL statement that accepts inputs or parameters at run time."
 
That's right, a bind variable is a simple placeholder; it's the equivalent of "insert your literal here".
Whenever you change a literal, it does not matter: the statement is effectively the same.
Similarly, the execution plan remains as is.
Since there is no reason for Oracle to do a hard parse of a statement it still has in the inventory, Oracle merely looks up the plan.

.. _`horse's mouth`: http://docs.oracle.com/database/121/TGSQL/tgsql_cursor.htm#TGSQL849



.. toctree::
   :hidden:

   PL/SQL Variables <plsql-variables>
   Bind Peeking <bind-peeking>
   Adaptive Cursor Sharing <adaptive-cursor-sharing>
   Generic Static Statements <smart-logic>