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

Most setups never hit this. By default, `CREATE SUBSCRIPTION` creates the
slot for you, and the publication must already exist because you name it
in the command. The order is right without you thinking about it.

You hit it only when you create the slot yourself. One common reason is a
large database. Copying big tables through the subscription can take hours,
so you restore the target from a snapshot of the source instead. The slot
must exist before you take the snapshot. Then it keeps every change written
after the snapshot, and the subscription replays them once it starts.

## Postgres 18 skips the changes instead

Up to Postgres 17, the subscription fails with the error above and retries
forever. Postgres 18 logs a warning and continues:

```
WARNING:  skipped loading publication "my_pub"
DETAIL:  The publication does not exist at this point in the WAL.
HINT:  Create the publication if it does not exist.
```

This is worse than the error. The changes written between the slot and the
publication never reach the target, and the subscription looks healthy.
I tested it on both versions: create the slot, insert row 1, create the
publication, insert row 2, then subscribe. Postgres 17 stops with the
error. Postgres 18 copies row 2 and silently loses row 1.

## References

- [Logical replication](https://www.postgresql.org/docs/current/logical-replication.html)
- [Logical decoding concepts](https://www.postgresql.org/docs/current/logicaldecoding-explanation.html)
- [Creating a logical replica from a snapshot in RDS Postgres](https://tech.instacart.com/creating-a-logical-replica-from-a-snapshot-in-rds-postgres-886d9d2c7343), Instacart: the full snapshot and restore procedure, publication first
- [The warning in Postgres 18's pgoutput](https://github.com/postgres/postgres/blob/REL_18_STABLE/src/backend/replication/pgoutput/pgoutput.c)
