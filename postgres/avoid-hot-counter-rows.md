# Don't update one counter row from many writers

## The problem

Many workers write events, and we want a count per account per day. The
obvious way is a trigger on `events` that keeps a stats table up to date:

```sql
CREATE FUNCTION count_event() RETURNS trigger AS $$
BEGIN
  INSERT INTO event_stats (account_id, day, events)
  VALUES (NEW.account_id, NEW.created_at::date, 1)
  ON CONFLICT (account_id, day)
  DO UPDATE SET events = event_stats.events + 1;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER count_event AFTER INSERT ON events
  FOR EACH ROW EXECUTE FUNCTION count_event();
```

The inserts into `events` don't block each other, but each one now also
updates a row in `event_stats`. An update locks the row until the
transaction commits. When many workers write events for the same account
on the same day, they all need the same stats row, so they wait in line,
one transaction at a time. Long transactions make the line longer.

I hit this at work. The stats rows became a lock hot spot, and writes to
the events table slowed down with them.

## Insert, then aggregate later

Keep the write path insert-only. Each worker inserts its row and nothing
else, so it has no existing row to wait for. Build the counts away from
the writers, and accept that they are a little behind:

- **A periodic job.** It sums the events written since its last run and
  adds them to `event_stats` with `INSERT ... ON CONFLICT DO UPDATE`. Only
  the job updates the stats rows, so nothing waits on them. This is what
  we did.
- **A materialized view.** Refresh it on a schedule with
  `REFRESH MATERIALIZED VIEW CONCURRENTLY`, which needs a unique index.
  Each refresh recomputes everything, so it suits small tables.
- **LISTEN/NOTIFY.** Each insert sends a `NOTIFY`, and a separate service
  adds up the counts in memory and writes them every few seconds. A
  notification reaches only connected listeners, so the service must
  rebuild from the events table when it starts.

## When it matters

A trigger like this is fine when writes spread over many stats rows. The
problem appears when many writers hit the same row: a busy account, a
global total, a daily counter.

[The Art of PostgreSQL](https://theartofpostgresql.com) by Dimitri
Fontaine calls this trigger "a classic anti-pattern". Its benchmark
measures the cost: 1,000 increments from 100 workers on one row took 3.1
seconds as updates and 2.1 seconds as inserts.
