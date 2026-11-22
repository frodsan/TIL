# Don't update aggregate tables from triggers

## The context

We had a ledger table where each entry has a `debit` and a `credit`. To
report totals by month or year, we didn't want to read every entry. So we
kept a summary table with one row per account per day, and summed those
rows instead.

The summary had to stay right, so every time an entry's `debit` or
`credit` changed, the summary row of its day changed too. We did that
with a trigger:

```sql
CREATE FUNCTION update_daily_stats() RETURNS trigger AS $$
BEGIN
  INSERT INTO daily_ledger_stats (account_id, day, debit, credit)
  VALUES (NEW.account_id, NEW.date,
          NEW.debit - coalesce(OLD.debit, 0),
          NEW.credit - coalesce(OLD.credit, 0))
  ON CONFLICT (account_id, day) DO UPDATE
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
each one into an update of a summary row. An update locks the row until
the transaction commits. When many transactions write entries for the
same account and day, they all need the same summary row, so they wait in
line, one at a time. A long transaction, such as a large import, holds
the lock for its whole length.

I tested it on Postgres 18. One transaction inserted an entry and stayed
open for 4 seconds. A second insert for the same account and day waited
3.1 seconds, until the first one committed. An insert for another account
finished in 0.1 seconds.

## The fix: insert, then aggregate later

Keep the write path free of shared rows. Each transaction writes only its
own entries, or inserts the change into a separate table, which never
waits on another row. A periodic job then sums the changes since its last
run and applies them to the summary with `INSERT ... ON CONFLICT DO
UPDATE`. Only the job updates the summary rows, so nothing waits on them.
The summary is a little behind, which is fine for monthly and yearly
reports.

Two other ways to build the summary away from the writers:

- **A materialized view**, refreshed on a schedule with
  `REFRESH MATERIALIZED VIEW CONCURRENTLY`. Each refresh recomputes
  everything, so it suits small tables.
- **LISTEN/NOTIFY.** Each write sends a `NOTIFY`, and a separate service
  adds up the changes in memory and writes them every few seconds. A
  notification reaches only connected listeners, so the service must
  rebuild from the ledger when it starts.

A trigger like this is fine when writes spread over many summary rows.
The problem appears when many writers hit the same row.

## References

- [The Art of PostgreSQL](https://theartofpostgresql.com), Dimitri
  Fontaine. Its chapter on triggers calls a trigger that turns inserts
  into updates of one row "a classic anti-pattern". Its benchmark shows
  the cost: 1,000 increments of one row from 100 workers took 3.1 seconds
  as updates and 2.1 seconds as inserts.
- [Explicit locking: row-level locks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS)
