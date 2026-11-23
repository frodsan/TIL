# NOT IN returns no rows when its list has a NULL

To find companies without ledger entries, this query looks right:

```sql
SELECT name FROM companies
WHERE id NOT IN (SELECT company_id FROM ledger_entries);
```

It works until one entry has a NULL `company_id`. Then it returns no rows
at all, without any error, as if every company had entries.

## Why

`id NOT IN (1, 2, NULL)` means `id <> 1 AND id <> 2 AND id <> NULL`. A
comparison with NULL is unknown, not true or false. An `AND` with an
unknown part can't be true, and `WHERE` keeps a row only when its
condition is true. So every row is dropped.

`IN` doesn't have this problem: `3 IN (1, 3, NULL)` is true, because one
match is enough.

## The fix

Use `NOT EXISTS`. It asks, for each company, whether a matching entry
exists, and a NULL `company_id` never matches:

```sql
SELECT name FROM companies c
WHERE NOT EXISTS (
  SELECT 1 FROM ledger_entries e WHERE e.company_id = c.id
);
```

It's also faster: Postgres plans it as an anti-join, which it can't do
for `NOT IN` because of the NULL rule.

`EXCEPT` works too, when you only need the ids:

```sql
SELECT id FROM companies
EXCEPT
SELECT company_id FROM ledger_entries;
```

Keep `NOT IN` for fixed lists you write yourself, like
`status NOT IN ('draft', 'void')`.

## References

1. [The Art of PostgreSQL](https://theartofpostgresql.com), Dimitri
   Fontaine. It recommends `NOT EXISTS` for anti-joins and notes that
   Postgres ignores what the subquery selects.
2. [What Is NULL?](https://thegresqlpost.org/presentations/what-is-null/2026-10-21-handout.pdf),
   Vik Fearing, PostgreSQL Conference Europe 2026. It shows that
   `NOT IN` means `<> ALL`, so one unknown comparison is enough to drop
   every row, and gives `NOT EXISTS` and `EXCEPT` as fixes.
