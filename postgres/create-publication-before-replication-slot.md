# Create the publication before the replication slot

Logical replication has three parts:

- A publication on the source database. It lists the tables to send.
- A replication slot on the source database. It keeps the WAL until the subscriber has read it.
- A subscription on the target database. It receives the changes and applies them.

I created the slot first and the publication a few seconds later. When I
enabled the subscription, it failed with this error, even though the
publication existed:

```
ERROR:  publication "my_pub" does not exist
```

## Why

The slot starts streaming from the WAL position where it was created.
Postgres decodes each change with the catalog as it was when that change
was written, not as it is now. The first changes after the slot came from
a moment when the publication did not exist yet, so the lookup failed.

## The fix

Create the publication first, then the slot:

```sql
-- On the source database
CREATE PUBLICATION my_pub FOR TABLE orders, order_items;
SELECT pg_create_logical_replication_slot('my_slot', 'pgoutput');

-- On the target database
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=source dbname=app user=replicator'
  PUBLICATION my_pub
  WITH (create_slot = false, slot_name = 'my_slot', copy_data = false);
```

When the subscription creates its own slot (the default), the order is
right by itself. The problem appears when you create the slot by hand, for
example to fix the starting point before you take a snapshot of the
source and restore the target from it.

Newer Postgres versions may skip the missing publication with a warning
instead of an error. Check the behavior on your version.

## References

- [Logical replication](https://www.postgresql.org/docs/current/logical-replication.html)
- [Logical decoding concepts](https://www.postgresql.org/docs/current/logicaldecoding-explanation.html)
