# Join elimination not possible when joining to security barrier view

## Description

1. When LEFT JOINing to a security barrier view on a unique index, join elimination is not performed.
2. When selecting all fields the security barrier view is still significantly slower than the normal view.

## Example

The following shows the issue on postgres 12.18 and 16.2.

```sql
-- setup
create table tabx as select * from generate_series(1,1e6,1e5) idx;
create table taby as select * from generate_series(1,1e6) idy;
create unique index on taby using btree (idy);
create view viewy_nobarrier as select * from taby;
create view viewy_barrier with (security_barrier=true) as select * from taby;

-- test (join elimination)
explain analyze select idx from tabx x left join viewy_nobarrier y on idy = idx; -- fast
explain analyze select idx from tabx x left join viewy_barrier y on idy = idx;   -- slow

-- test (select *)
explain analyze select * from tabx x left join viewy_nobarrier y on idy = idx; -- fast
explain analyze select * from tabx x left join viewy_barrier y on idy = idx;   -- slow
```

## Comments

Usually I'd start by identifying all function calls (including operators) and checking if any of them are missing the LEAKPROOF property.
I didn't find any here (though I could have missed them).

The slow plan shows Sort and Materialize nodes which are not present in the fast plan.
This makes me wonder if its related to another issue we found where it seemed like ORDER BY couldn't be pushed down into a security barrier view, also resulting in a Materialize node iirc.

## Plans (for convenience)

These plans are generated on postgres 16.2.

```sql
postgres=# explain analyze select idx from tabx x left join viewy_nobarrier y on idy = idx; -- fast
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Seq Scan on tabx x  (cost=0.00..23.60 rows=1360 width=32) (actual time=0.007..0.008 rows=10 loops=1)
 Planning Time: 0.049 ms
 Execution Time: 0.016 ms
(3 rows)

postgres=# explain analyze select idx from tabx x left join viewy_barrier y on idy = idx;   -- slow
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Merge Left Join  (cost=138158.23..242665.03 rows=6800000 width=32) (actual time=148.182..421.900 rows=10 loops=1)
   Merge Cond: (x.idx = taby.idy)
   ->  Sort  (cost=94.38..97.78 rows=1360 width=32) (actual time=0.020..0.023 rows=10 loops=1)
         Sort Key: x.idx
         Sort Method: quicksort  Memory: 25kB
         ->  Seq Scan on tabx x  (cost=0.00..23.60 rows=1360 width=32) (actual time=0.010..0.012 rows=10 loops=1)
   ->  Materialize  (cost=138063.84..143063.84 rows=1000000 width=32) (actual time=145.640..328.904 rows=900002 loops=1)
         ->  Sort  (cost=138063.84..140563.84 rows=1000000 width=32) (actual time=145.635..246.236 rows=900002 loops=1)
               Sort Key: taby.idy
               Sort Method: external merge  Disk: 10768kB
               ->  Seq Scan on taby  (cost=0.00..14480.00 rows=1000000 width=32) (actual time=0.025..50.121 rows=1000000 loops=1)
 Planning Time: 0.125 ms
 JIT:
   Functions: 6
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.203 ms, Inlining 0.000 ms, Optimization 0.147 ms, Emission 2.364 ms, Total 2.714 ms
 Execution Time: 423.326 ms
(17 rows)
```
