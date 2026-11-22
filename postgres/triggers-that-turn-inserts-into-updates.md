# Triggers that turn inserts into updates cause lock contention

At Pennylane, we had a ledger table where each entry has a `debit` and a
`credit`. To report totals by month or year, we didn't want to read every
entry. So we kept an aggregate table with one row per company and date,
and summed those rows instead.

The aggregate table had to stay right, so a trigger on `INSERT`, `UPDATE`
and `DELETE` added each change to the row of its date. For an update:

```sql
CREATE FUNCTION update_ledger_entries_stats() RETURNS trigger AS $$
BEGIN
  INSERT INTO ledger_entries_stats (company_id, date, debit, credit)
  VALUES (NEW.company_id, NEW.date,
          NEW.debit - OLD.debit, NEW.credit - OLD.credit)
  ON CONFLICT (company_id, date) DO UPDATE
  SET debit = ledger_entries_stats.debit + excluded.debit,
      credit = ledger_entries_stats.credit + excluded.credit;
  RETURN NULL;
END $$ LANGUAGE plpgsql;
```

## The problem

Writes to `ledger_entries` don't block each other, but the trigger turns
each one into an update of the aggregate table. An update locks the row
until the transaction commits. When many transactions write entries for
the same company and date, they all need the same row in the aggregate
table, so they wait in line, one at a time. A long transaction, such as a
large import, holds the lock for its whole length.

## The fix: insert the changes, aggregate later

We kept the trigger, but it now only inserts each change
into a deltas table. An insert adds the new values, a delete adds the old
values negated, and an update does both:

```sql
CREATE FUNCTION record_ledger_delta() RETURNS trigger AS $$
BEGIN
  IF TG_OP IN ('UPDATE', 'DELETE') THEN
    INSERT INTO ledger_entries_stats_deltas (company_id, date, debit, credit)
    VALUES (OLD.company_id, OLD.date, -OLD.debit, -OLD.credit);
  END IF;
  IF TG_OP IN ('INSERT', 'UPDATE') THEN
    INSERT INTO ledger_entries_stats_deltas (company_id, date, debit, credit)
    VALUES (NEW.company_id, NEW.date, NEW.debit, NEW.credit);
  END IF;
  RETURN NULL;
END $$ LANGUAGE plpgsql;
```

Inserts never wait on each other. Later, outside the writing transaction,
a job or service flushes the deltas into the aggregate table:

```sql
WITH batch AS (
  DELETE FROM ledger_entries_stats_deltas
  WHERE id IN (
    SELECT id FROM ledger_entries_stats_deltas
    WHERE company_id = $1
    FOR UPDATE SKIP LOCKED
  )
  RETURNING *
)
INSERT INTO ledger_entries_stats (company_id, date, debit, credit)
SELECT company_id, date, sum(debit), sum(credit)
FROM batch
GROUP BY company_id, date
ON CONFLICT (company_id, date) DO UPDATE
SET debit = ledger_entries_stats.debit + excluded.debit,
    credit = ledger_entries_stats.credit + excluded.credit;
```

Two other ways to build the aggregate table away from the writers:

- **A materialized view**, refreshed on a schedule with
  `REFRESH MATERIALIZED VIEW CONCURRENTLY`. Each refresh recomputes
  everything, so it suits small tables.
- **LISTEN/NOTIFY.** Each write sends a `NOTIFY`, and a separate service
  adds up the changes in memory and writes them every few seconds
  [[1]](#references).

## References

1. [The Art of PostgreSQL](https://theartofpostgresql.com), Dimitri
   Fontaine, chapter "Listen and Notify". The service runs `LISTEN`
   first, rebuilds its totals from the source table, then adds each
   notification to them in memory and saves them with one upsert every
   few seconds. Its "Triggers" chapter calls a trigger that turns inserts
   into updates of one row "a classic anti-pattern".
2. [Explicit locking: row-level locks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS)
