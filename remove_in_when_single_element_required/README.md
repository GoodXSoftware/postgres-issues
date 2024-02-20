# Remove `x IN (s)` when single element of `s` is already checked

## Description

A WHERE clause containing `x IN (s) AND x = y` should be simplified to `x = y` when `y IN (s)`.

## Use case

We have views that restrict the values of a column based on the application user logged in.
These views are typically defined as:
```sql
create view foo as select * from bar where entity = any ((select * from allowed_entities_for_user()));
```
Users then query on this view like:
```sql
select * from foo where entity = 1;
```
The rewritten query becomes:
```sql
select * from bar where entity = any ((select * from allowed_entities_for_user())) and entity = y;
```
It is possible that the user has access to many entities, up to a couple hundred.
If it can be determined that `allowed_entities_for_user()` contains `y`, then this should be simplified to:
```sql
select * from bar where entity = y;
```
This generally seems to work as expected, but not when querying security barrier views.

## Example

```sql
-- setup
create table foo as select id from (select s % 1000 as id from generate_series(1, 1e6) s) x;
create index on foo using btree (id);
create view bar with(security_barrier=true) as select * from foo where id in (select * from generate_series(1, 500));

-- This works as expected.
explain analyze select * from foo where id = 400;
--                                                         QUERY PLAN
-- --------------------------------------------------------------------------------------------------------------------------
--  Bitmap Heap Scan on foo  (cost=95.17..4793.05 rows=5000 width=32) (actual time=0.170..0.715 rows=1000 loops=1)
--    Recheck Cond: (id = '400'::numeric)
--    Heap Blocks: exact=1000
--    ->  Bitmap Index Scan on foo_id_idx  (cost=0.00..93.92 rows=5000 width=0) (actual time=0.094..0.094 rows=1000 loops=1)
--          Index Cond: (id = '400'::numeric)
--  Planning Time: 0.043 ms
--  Execution Time: 0.740 ms

-- This works as expected.
explain analyze select * from foo where id in (select * from generate_series(1, 500)) and id = 400;
--                                                               QUERY PLAN
-- --------------------------------------------------------------------------------------------------------------------------------------
--  Nested Loop  (cost=102.69..4938.08 rows=5000 width=32) (actual time=0.202..0.768 rows=1000 loops=1)
--    ->  Unique  (cost=7.51..7.52 rows=2 width=4) (actual time=0.045..0.046 rows=1 loops=1)
--          ->  Sort  (cost=7.51..7.52 rows=2 width=4) (actual time=0.045..0.046 rows=1 loops=1)
--                Sort Key: ((generate_series.generate_series)::numeric)
--                Sort Method: quicksort  Memory: 25kB
--                ->  Function Scan on generate_series  (cost=0.00..7.50 rows=2 width=4) (actual time=0.037..0.042 rows=1 loops=1)
--                      Filter: ((generate_series)::numeric = '400'::numeric)
--                      Rows Removed by Filter: 499
--    ->  Materialize  (cost=95.17..4818.05 rows=5000 width=32) (actual time=0.156..0.682 rows=1000 loops=1)
--          ->  Bitmap Heap Scan on foo  (cost=95.17..4793.05 rows=5000 width=32) (actual time=0.156..0.635 rows=1000 loops=1)
--                Recheck Cond: (id = '400'::numeric)
--                Heap Blocks: exact=1000
--                ->  Bitmap Index Scan on foo_id_idx  (cost=0.00..93.92 rows=5000 width=0) (actual time=0.087..0.087 rows=1000 loops=1)
--                      Index Cond: (id = '400'::numeric)
--  Planning Time: 0.067 ms
--  Execution Time: 0.803 ms

-- This is slow.
explain analyze select * from bar where id = 400;
--                                                                    QUERY PLAN
-- ------------------------------------------------------------------------------------------------------------------------------------------------
--  Gather  (cost=1010.75..19800.34 rows=2500 width=4) (actual time=0.452..69.431 rows=1000 loops=1)
--    Workers Planned: 2
--    Workers Launched: 2
--    ->  Subquery Scan on bar  (cost=10.75..18550.34 rows=2500 width=4) (actual time=0.431..65.093 rows=333 loops=3)
--          Filter: (bar.id = '400'::numeric)
--          Rows Removed by Filter: 166333
--          ->  Hash Join  (cost=10.75..12300.34 rows=208333 width=4) (actual time=0.315..55.473 rows=166667 loops=3)
--                Hash Cond: (foo.id = (generate_series.generate_series)::numeric)
--                ->  Parallel Seq Scan on foo  (cost=0.00..8591.67 rows=416667 width=4) (actual time=0.022..15.179 rows=333333 loops=3)
--                ->  Hash  (cost=8.25..8.25 rows=200 width=4) (actual time=0.278..0.279 rows=500 loops=3)
--                      Buckets: 1024  Batches: 1  Memory Usage: 26kB
--                      ->  HashAggregate  (cost=6.25..8.25 rows=200 width=4) (actual time=0.180..0.215 rows=500 loops=3)
--                            Group Key: (generate_series.generate_series)::numeric
--                            ->  Function Scan on generate_series  (cost=0.00..5.00 rows=500 width=4) (actual time=0.031..0.078 rows=500 loops=3)
--  Planning Time: 0.097 ms
--  Execution Time: 69.473 ms
```
