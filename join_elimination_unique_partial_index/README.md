# Join elimination not possible with unique partial index

## Description

When LEFT JOINing to a table on a unique partial index, join elimination is not performed.

Jacques already reported this to Hans in late 2023.

## Example

The following shows the error on postgres 12.18 and 16.2.

```sql
-- setup
create table foo as select * from (values (1, 5, true), (8, 3, true)) x(f1, f2, factive);
create table links (l1 int not null, l2 int not null, lactive bool not null);
insert into links (l1, l2, lactive) select s1, s2, s3 = 2 from generate_series(1, 100) s1, generate_series(1, 100) s2, generate_series(1, 2) s3;

-- test (no indexes, works as expected)
explain select foo.* from foo left join links l on l1 = f1 and l2 = f2 and lactive;
--                                QUERY PLAN
-- -------------------------------------------------------------------------
--  Merge Left Join  (cost=1315.23..1428.25 rows=2200 width=9)
--    Merge Cond: ((foo.f1 = l.l1) AND (foo.f2 = l.l2))
--    ->  Sort  (cost=154.14..159.64 rows=2200 width=9)
--          Sort Key: foo.f1, foo.f2
--          ->  Seq Scan on foo  (cost=0.00..32.00 rows=2200 width=9)
--    ->  Sort  (cost=1161.10..1191.07 rows=11990 width=8)
--          Sort Key: l.l1, l.l2
--          ->  Seq Scan on links l  (cost=0.00..348.80 rows=11990 width=8)
--                Filter: lactive

-- test (unique partial index, expect join elimination but not being done)
create unique index on links using btree (l1, l2) where lactive;
explain select foo.* from foo left join links on l1 = f1 and l2 = f2 and lactive;
--                                           QUERY PLAN
-- ----------------------------------------------------------------------------------------------
--  Merge Right Join  (cost=154.42..932.42 rows=2200 width=9)
--    Merge Cond: ((links.l1 = foo.f1) AND (links.l2 = foo.f2))
--    ->  Index Only Scan using links_l1_l2_idx on links  (cost=0.29..706.28 rows=10000 width=8)
--    ->  Sort  (cost=154.14..159.64 rows=2200 width=9)
--          Sort Key: foo.f1, foo.f2
--          ->  Seq Scan on foo  (cost=0.00..32.00 rows=2200 width=9)

-- test (normal unique index, works as expected)
create unique index on links using btree (l1, l2, lactive);
explain select foo.* from foo left join links on l1 = f1 and l2 = f2 and lactive = factive;
--                       QUERY PLAN
-- -------------------------------------------------------
--  Seq Scan on foo  (cost=0.00..32.00 rows=2200 width=9)
```

## Comments

Hans said this in a recent email - not sure if he was referring to this issue specifically:
>as for the things brought up to Jacques it seems those were not bugs but had a reason why it had to be that way. \
i don’t recall the details but i can certainly look it up. i remember checking those things with Laurenz.

Unless I'm missing something (which is definitely possible), if a unique index is created on `x WHERE y`, and the join condition is `LEFT JOIN foo ON x = ... AND y`, then join elimination should be possible.
Of course the same cannot be said for the case where the join condition is just `LEFT JOIN foo ON x = ...` (without `y`).
