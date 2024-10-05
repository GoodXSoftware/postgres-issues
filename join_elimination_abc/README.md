# Join elimination not performed correctly in edge case

https://www.postgresql.org/message-id/63490.1714386843@antos

## Description

Join elimination is not performed correctly in an edge case.
This is best demonstrated at the hand of the following example.

## Example

The following shows the issue on postgres 16.2.

```sql
-- create 3 identical tables
create table a as select id1, id2 from generate_series(1, 100) id1, generate_series(1, 100) id2;
create table b as select id1, id2 from generate_series(1, 100) id1, generate_series(1, 100) id2;
create table c as select id1, id2 from generate_series(1, 100) id1, generate_series(1, 100) id2;
alter table a add unique (id1, id2);
alter table b add unique (id1, id2);
alter table c add unique (id1, id2);

-- join elimination works as expected
explain (costs off)
  select a.*
    from a
      left join b on (b.id1, b.id2) = (a.id1, a.id2)
      left join c on (c.id1, c.id2) = (a.id1, a.id2);
--   QUERY PLAN
-- ---------------
--  Seq Scan on a

-- join elimination works as expected
explain (costs off)
  select a.*
    from a
      left join b on (b.id1, b.id2) = (a.id1, a.id2)
      left join c on (c.id1, c.id2) = (b.id1, b.id2);
--   QUERY PLAN
-- ---------------
--  Seq Scan on a

-- join elimination fails
-- expect both b and c to be eliminated, but b remains
explain (costs off)
  select a.*
    from a
      left join b on (b.id1, b.id2) = (a.id1, a.id2)
      left join c on (c.id1, c.id2) = (a.id1, b.id2);
--                      QUERY PLAN
-- ----------------------------------------------------
--  Hash Left Join
--    Hash Cond: ((a.id1 = b.id1) AND (a.id2 = b.id2))
--    ->  Seq Scan on a
--    ->  Hash
--          ->  Seq Scan on b

-- join elimination works as expected
-- when c is manually removed from query, join elimination works for b
explain (costs off)
  select a.*
    from a
      left join b on (b.id1, b.id2) = (a.id1, a.id2);
--   QUERY PLAN
-- ---------------
--  Seq Scan on a
```
