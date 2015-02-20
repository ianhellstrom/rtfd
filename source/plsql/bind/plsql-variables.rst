.. _plsql-bind-variables:
 
PL/SQL Variables
================
Bind variables are related to host variables.
Host variables are defined in the host or caller, whereas bind variables accept values from the caller to SQL.
In PL/SQL the distinction between bind and host variables disappears.
 
Bind variables can be used in almost any place in PL/SQL with one exception: bind variables in anonymous PL/SQL blocks cannot appear in conditional compilation directives.
 
Variables in ``WHERE`` and ``VALUES`` clauses of *static* DML statements are automatically made bind variables in PL/SQL.
Before you run off and think that Oracle takes care of everything for you, hang on a minute.
That only applies to static statements; the emphasis in the previous sentence was intentional.
Dynamic SQL is slightly different.
 
Let's take a look at five common examples:
 
.. code-block:: sql
   :linenos:
 
    --
    -- Example 1: typically found in anonymous PL/SQL blocks
    --
    SELECT
      *
    BULK COLLECT INTO
      ...
    FROM
      fridge
    WHERE
      product = 'Beer';                -- hard-coded literal
 
    --
    -- Example 2: typically found in anonymous PL/SQL blocks
    --
    SELECT
      *
    BULK COLLECT INTO
      ...
    FROM
      fridge
    WHERE
      product = :prod;                 -- bind variable
 
    --
    -- Example 3: typically found in PL/SQL modules
    --
    SELECT
      *
    BULK COLLECT INTO
      ...
    FROM
      fridge
    WHERE
      product = product_in;            -- bind variable
 
    --
    -- Example 4: typically found in PL/SQL modules
    --
    EXECUTE IMMEDIATE
      'SELECT * FROM fridge ' ||
      'WHERE product = ''' ||
      product_in || ''''               -- literal: value is embedded in statement
      BULK COLLECT INTO ...
    ;          
 
    --
    -- Example 5: typically found in PL/SQL modules
    --
    EXECUTE IMMEDIATE
      'SELECT * FROM fridge ' ||
      'WHERE product = :prod'          -- bind variable, because...
      BULK COLLECT INTO ...
      USING product_in                 -- ... we substitute rather than embed the value
    ;
 
Hopefully, no one would build dynamic SQL like that as it is open to SQL injections; the package ``DBMS_ASSERT`` offers some basic sanity checks on raw (user) input.
The code is only shown for the purposes of our demonstration of the various options with regard to literals and bind variables.
You can  use ``DBMS_SQL`` as an alternative to dynamically build a SQL statement, but we have decided not to show the code for reasons of brevity.
 
There is sometimes `a good reason`_ to utilize ``DBMS_SQL`` instead of native dynamic SQL (NDS).
NDS has to be parsed every time the statement is executed; for complex statements the overhead can be significant.
Parsing can be bypassed with ``DBMS_SQL``.
For instance, when a statement is executed for different values inside a loop, you just have to place the call to ``DBMS_SQL.PARSE`` outside of the loop; the calls to ``DBMS_SQL.BIND_VARIABLE`` need to be placed inside the loop.

.. _`a good reason`: http://www.oracle.com/technetwork/issue-archive/o64sql-095035.html