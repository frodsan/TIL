# Don't update one counter row from many writers

## The problem

The obvious way to count retweets is a column on the tweet:

```sql
UPDATE tweets SET retweets = retweets + 1 WHERE id = 3;
```

An update locks the row until the transaction commits. When 100 users
retweet tweet 3 at the same moment, the first update takes the lock and
the other 99 wait in line. The work is tiny, but it runs one at a time.

## Insert one row per action instead

```sql
INSERT INTO tweet_activity (tweet_id, action) VALUES (3, 'retweet');
```

An insert writes a new row, so it has no row to wait for. The 100 inserts
run side by side. In the benchmark in
[The Art of PostgreSQL](https://theartofpostgresql.com), 1,000 retweets
from 100 workers took 3.1 seconds with updates and 2.1 seconds with
inserts.

## Don't sum it back with a trigger

Counting the rows on every read is slow, so the next idea is a trigger
that adds 1 to a summary row on each insert. That brings the problem back:
every insert now updates the same summary row, and the writers wait in
line again. The book calls it "a classic anti-pattern".

I hit this at work. A trigger kept a stats table up to date from an
events table, and the stats rows became a lock hot spot. We replaced the
trigger with an insert-only table and a job that aggregates it.

## Alternatives

Update the summary away from the writers, and accept that it is a few
seconds behind:

- **A periodic job.** It sums the new activity rows and adds the result
  to the summary with `INSERT ... ON CONFLICT DO UPDATE`.
- **A materialized view.** Refresh it on a schedule with
  `REFRESH MATERIALIZED VIEW CONCURRENTLY`, which needs a unique index.
- **LISTEN/NOTIFY.** Each insert sends a `NOTIFY`, and a separate service
  adds up the counts in memory and writes them every few seconds. A
  notification reaches only connected listeners, so the service must
  rebuild from the activity table when it starts.

A plain counter column is fine when writes spread over many rows, such as
one row per user. The problem appears only when many writers hit the same
row: a popular post, a global stock level, a daily total.
