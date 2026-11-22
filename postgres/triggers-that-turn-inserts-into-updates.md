# Triggers that turn inserts into updates cause lock contention

At Pennylane, we had a ledger table where each entry has a `debit` and a
`credit`. To report totals by month or year, we didn't want
to read every entry. So we kept an aggregate table with one row per
company per day, and summed those rows instead.

The aggregate table had to stay right, so every time an entry's `debit`
or `credit` changed, the row of its day in the aggregate table changed
too. We did that with a trigger:

```sql
CREATE FUNCTION update_daily_stats() RETURNS trigger AS $$
BEGIN
  INSERT INTO daily_ledger_stats (company_id, day, debit, credit)
  VALUES (NEW.company_id, NEW.date,
          NEW.debit - coalesce(OLD.debit, 0),
          NEW.credit - coalesce(OLD.credit, 0))
  ON CONFLICT (company_id, day) DO UPDATE
  SET debit = daily_ledger_stats.debit + excluded.debit,
      credit = daily_ledger_stats.credit + excluded.credit;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER update_daily_stats
  AFTER INSERT OR UPDATE OF debit, credit ON ledger_entries
  FOR EACH ROW EXECUTE FUNCTION update_daily_stats();
```

## The problem

Writes to `ledger_entries` don't block each other, but the trigger turns
each one into an update of the aggregate table. An update locks the row
until the transaction commits. When many transactions write entries for
the same company and day, they all need the same row in the aggregate
table, so they wait in
line, one at a time. A long transaction, such as a large import, holds
the lock for its whole length.

## The fix: insert, then aggregate later

Keep the write path free of shared rows. Each transaction writes only its
own entries, or inserts the change into a separate table, which never
waits on another row. A periodic job then sums the changes since its last
run and applies them to the aggregate table with `INSERT ... ON CONFLICT
DO UPDATE`. Only the job updates the aggregate table, so nothing waits on
it. The aggregate table is a little behind, which is fine for monthly and
yearly reports.

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
