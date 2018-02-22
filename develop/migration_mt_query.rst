.. _mt_query_migration:

Multi-Tenant Query Migration
============================

To execute queries efficiently for a specific tenant Citus needs to route them to the appropriate node and run them there. Thus every query must identify which tenant it involves. For simple select, update, and delete queries this means that the *where* clause must filter by tenant id.

Suppose we want to get the details for an order. It used to suffice to filter by order_id. However once orders are distributed by store_id we must include that in the where filter as well.

.. code-block:: sql

  -- before
  SELECT * FROM orders WHERE order_id = 123;

  -- after
  SELECT * FROM orders WHERE order_id = 123 AND store_id = 42;

Likewise insert statements must always include a value for the tenant id column. Citus inspects that value for routing the insert command.

When joining tables make sure to filter by tenant id. For instance here is how to inspect how many awesome wool pants a given store has sold:

.. code-block:: sql

  -- One way is to include store_id in the join and also
  -- filter by it in one of the queries

  SELECT sum(l.quantity)
    FROM line_items l
   INNER JOIN products p
      ON l.product_id = p.product_id
     AND l.store_id = p.store_id
   WHERE p.name='Awesome Wool Pants'
     AND l.store_id='8c69aa0d-3f13-4440-86ca-443566c1fc75'

  -- Equivalently you omit store_id from the join condition
  -- but filter both tables by it. This may be useful if
  -- building the query in an ORM

  SELECT sum(l.quantity)
    FROM line_items l
   INNER JOIN products p ON l.product_id = p.product_id
   WHERE p.name='Awesome Wool Pants'
     AND l.store_id='8c69aa0d-3f13-4440-86ca-443566c1fc75'
     AND p.store_id='8c69aa0d-3f13-4440-86ca-443566c1fc75'

Validating Query Migration
--------------------------

With large and complex application code-bases, certain queries generated by the application can often be overlooked, and thus won't have a tenant_id filter on them. Citus' parallel executor will still execute these queries successfully, and so, during testing, these queries remain hidden since the application still works fine. However, if a query doesn't contain the tenant_id filter, Citus' executor will hit every shard in parallel, but only one will return any data.  This consumes resources needlessly, and may exhibit itself as a problem only when one moves to a higher-throughput production environment.

To prevent encoutering such issues only after launching in production, one can set a config value to log queries which hit more than one shard. In a properly configured and migrated multi-tenant application, each query should only hit one shard at a time.

During testing, one can configure the following:

.. code-block:: sql

  -- adjust for your own database's name of course

  ALTER DATABASE citus SET citus.multi_task_query_log_level = 'error';

Citus will then error out if it encounters queries which are going to hit more than one shard. Erroring out during testing allows the application developer to find and migrate such queries.

During a production launch, one can configure the same setting to log, instead of error out:

.. code-block:: sql

  ALTER DATABASE citus SET citus.multi_task_query_log_level = 'log';

The :ref:`configuration parameter section <multi_task_logging>` has more info on supported values for this setting.