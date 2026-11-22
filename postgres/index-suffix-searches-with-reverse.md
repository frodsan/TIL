# Index suffix searches with reverse()

A B-tree index can serve a prefix search like `email LIKE 'jane%'`, but
not a suffix search like `email LIKE '%@bigcorp.com'`. A suffix search
reads the whole table.

The trick is to reverse the column. A suffix of `email` is a prefix of
`reverse(email)`, so an index on the reversed value turns the suffix
search into a prefix search:

```sql
CREATE INDEX users_email_rev
  ON users (reverse(email) text_pattern_ops);

SELECT *
FROM users
WHERE reverse(email) LIKE reverse('%@bigcorp.com');
```

`reverse('%@bigcorp.com')` is `'moc.procgib@%'`, which ends in `%`, so
the index can find it. Postgres computes it once, when it plans the query.
The query must use the same expression as the index, `reverse(email)`.

## Don't forget text_pattern_ops

Most databases use a locale collation such as `en_US.UTF-8`. With one, a
plain B-tree index can't serve `LIKE 'abc%'`, because it sorts text by
locale rules, not byte by byte. `text_pattern_ops` builds the index in
byte order, which is what `LIKE` needs. A database with the `C` collation
doesn't need it.

I tested it on Postgres 18 with an `en_US.utf8` database and 200,000
emails, searching for `'%@tiny.io'`. With
`CREATE INDEX ON users (reverse(email))`, the query still read the whole
table, even with `enable_seqscan = off`. With `text_pattern_ops`, it used
the index:

```
Bitmap Index Scan on users_email_rev
  Index Cond: ((reverse(email) ~>=~ 'oi.ynit@'::text)
           AND (reverse(email) ~<~ 'oi.ynitA'::text))
```

## When to use something else

This only covers suffixes. For a search anywhere in the text, such as
`LIKE '%corp%'`, use a trigram index from the `pg_trgm` extension.

## References

- [Understanding the power of data types](https://postgres-data-types.pvh.ca/#13), Peter van Hardenberg: the talk with this tip
- [Operator classes and operator families](https://www.postgresql.org/docs/current/indexes-opclass.html): why `text_pattern_ops` exists
- [pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html)
