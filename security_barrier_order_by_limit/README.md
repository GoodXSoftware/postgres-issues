# Security barrier views and ORDER BY/LIMIT don't play nice

## Description

Using ORDER BY/LIMIT with a security barrier view can perform significantly slower than its non-barrier counterpart.

## Example

The following shows the issue on postgres 16.2.

```sql
-- setup
create table foo (id int, other int);
insert into foo (id, other) select s, s % 1e4 from generate_series(1, 1e6) s;
create index on foo using btree (id);
create index on foo using btree (other);
analyze foo;

create function slow()
 returns int
 language sql
 stable leakproof
as $$
    select pg_sleep(0.001);
    select 1;
$$;

create view nobarrier as select id, other, slow() from foo;
create view barrier with (security_barrier=true) as select id, other, slow() from foo;

-- identical plans when limit is omitted.
explain analyze select * from nobarrier where other = 42 order by id;
explain analyze select * from barrier where other = 42 order by id;
--                                                            QUERY PLAN
-- --------------------------------------------------------------------------------------------------------------------------------
--  Sort  (cost=386.35..386.60 rows=100 width=12) (actual time=209.675..209.678 rows=100 loops=1)
--    Sort Key: foo.id
--    Sort Method: quicksort  Memory: 28kB
--    ->  Bitmap Heap Scan on foo  (cost=5.20..383.03 rows=100 width=12) (actual time=2.246..209.607 rows=100 loops=1)
--          Recheck Cond: (other = 42)
--          Heap Blocks: exact=100
--          ->  Bitmap Index Scan on foo_other_idx  (cost=0.00..5.17 rows=100 width=0) (actual time=0.013..0.013 rows=100 loops=1)
--                Index Cond: (other = 42)
--  Planning Time: 0.073 ms
--  Execution Time: 209.697 ms


-- different plans when limit is specified.
explain analyze select * from nobarrier where other = 42 order by id limit 10; -- fast
--                                                                  QUERY PLAN
-- --------------------------------------------------------------------------------------------------------------------------------------------
--  Limit  (cost=360.19..362.81 rows=10 width=12) (actual time=2.281..18.965 rows=10 loops=1)
--    ->  Result  (cost=360.19..386.44 rows=100 width=12) (actual time=2.279..18.959 rows=10 loops=1)
--          ->  Sort  (cost=360.19..360.44 rows=100 width=8) (actual time=0.143..0.148 rows=10 loops=1)
--                Sort Key: foo.id
--                Sort Method: top-N heapsort  Memory: 25kB
--                ->  Bitmap Heap Scan on foo  (cost=5.20..358.03 rows=100 width=8) (actual time=0.033..0.134 rows=100 loops=1)
--                      Recheck Cond: (other = 42)
--                      Heap Blocks: exact=100
--                      ->  Bitmap Index Scan on foo_other_idx  (cost=0.00..5.17 rows=100 width=0) (actual time=0.019..0.021 rows=100 loops=1)
--                            Index Cond: (other = 42)
--  Planning Time: 0.120 ms
--  Execution Time: 18.991 ms

explain analyze select * from barrier where other = 42 order by id limit 10;   -- slow
--                                                               QUERY PLAN
-- --------------------------------------------------------------------------------------------------------------------------------------
--  Limit  (cost=385.19..385.21 rows=10 width=12) (actual time=199.314..199.317 rows=10 loops=1)
--    ->  Sort  (cost=385.19..385.44 rows=100 width=12) (actual time=199.311..199.313 rows=10 loops=1)
--          Sort Key: foo.id
--          Sort Method: top-N heapsort  Memory: 25kB
--          ->  Bitmap Heap Scan on foo  (cost=5.20..383.03 rows=100 width=12) (actual time=2.150..199.223 rows=100 loops=1)
--                Recheck Cond: (other = 42)
--                Heap Blocks: exact=100
--                ->  Bitmap Index Scan on foo_other_idx  (cost=0.00..5.17 rows=100 width=0) (actual time=0.014..0.014 rows=100 loops=1)
--                      Index Cond: (other = 42)
--  Planning Time: 0.075 ms
--  Execution Time: 199.335 ms
```

## Comments

It seems as though the security barrier materializes the entire resultset before applying the limit, whereas the nonbarrier view can apply the limit earlier and only materialize the final 10 records.

## Internal Note

This issue can be seen on telemedlive after login with the following query.
`bennie.debtor` is the debtor resource with barrier added back.
The query suddenly becomes fast when any of these changes are made:
1. Remove the line `patients,`.
2. Change `bennie.debtor` to `api.debtor` (remove the barrier).
3. Remove the `ORDER BY`.
When the `LIMIT` is removed, the barrier/nobarrier versions of the query are equally slow.
```sql
EXPLAIN ANALYZE
SELECT
    patients,
    uid
FROM bennie.debtor
WHERE uid = ANY (array(SELECT * FROM generate_series(1, 50000, 10)))
ORDER BY id
LIMIT 10;
```
