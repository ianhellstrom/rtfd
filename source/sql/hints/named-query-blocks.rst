.. _sql-hints-named-qb:
 
Named Query Blocks
==================
You may have already seen object aliases in the execution plan.
An object alias is a concatenation of the name of an object or its alias and the name of the query block it appears in.
A `query block`_ is any inline view or subquery of a SQL statement.
 
Object aliases typically look something like ``tab_name@SEL$1``, ``tab_nameINS$2``, ``tab_nameUPD$3``, ``tab_name@DEL$4``, or ``tab_name@MISC$5`` .
These automatically generated names are hardly insightful, which is why you are allowed to name query blocks yourself.
 
You name query blocks with ``/*+ QB_NAME( your_custom_qb_name ) */``.
Afterwards you can reference objects from that named query block using ``@your_custom_qb_name  tab_name_or_alias``.
The optimizer will use the custom name instead of ``SEL$1`` or whatever is applicable, so you can more easily understand the execution plan's details.

.. note::
   The optimizer ignores any hints that reference different query blocks with the same name.
 
Should you name all query blocks?
 
Hell no!
Only use the query block name hint when your statements are complex and you need to reference objects from various query blocks in your hints.
When would you want to do that?
When you use global hints.

.. _`query block`: http://jonathanlewis.wordpress.com/2007/06/25/qb_name
