# Indexes not used on inverse CASE

## Description

Consider the case where arbitray integer values are stored internally in tables, with a view that maps those integer values to text values which are more user-friendly, using a CASE.
Users query on the text values (the result of the CASE expression) in the view instead of the integer values in the tables.
When there is a 1-1 mapping of table values to view values it should be possible to use indexes by applying the inverse of the CASE.

## Example

The following shows the issue on postgres 12.18 and 16.2.

```sql
-- create a bunch of records, with code being one of (0,1,2,3,4).
create table ugly as select s as id, s % 5 as code from generate_series(1, 1e6) s;
create index on ugly using btree (code);

-- create the view with the values remapped.
create view pretty as
        select id,
        case code
            when 0 then 'c0'
            when 1 then 'c1'
            when 2 then 'c2'
            when 3 then 'c3'
            when 4 then 'c4'
        end as name
    from ugly;

-- This uses the index as expected.
explain analyze select * from ugly where code = 0;       -- fast
--                                      QUERY PLAN
-- ------------------------------------------------------------------------------------
--  Bitmap Heap Scan on ugly  (cost=3837.30..11833.55 rows=204500 width=10)
--    Recheck Cond: (code = '0'::numeric)
--    ->  Bitmap Index Scan on ugly_code_idx  (cost=0.00..3786.18 rows=204500 width=0)
--          Index Cond: (code = '0'::numeric)


-- This does not use the index, but could if it had rewritten `name = 'c0'` to `code = 0`.
explain analyze select * from pretty where name = 'c0';  -- slow
--                                                                                                                QUERY PLAN                                                                                                  
-- ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--  Gather  (cost=1000.00..17382.70 rows=5000 width=38)
--    Workers Planned: 2
--    ->  Parallel Seq Scan on ugly  (cost=0.00..15882.70 rows=2083 width=38)
--          Filter: (CASE code WHEN '0'::numeric THEN 'c0'::text WHEN '1'::numeric THEN 'c1'::text WHEN '2'::numeric THEN 'c2'::text WHEN '3'::numeric THEN 'c3'::text WHEN '4'::numeric THEN 'c4'::text ELSE NULL::text END = 'c0'::text)
```

## Comments

Disclaimer: I have zero experience on proving transforms.

Nevertheless, I imagine something like this transform should work:
```sql
(CASE WHEN x1 THEN y1 WHEN x2 THEN y2 ... END) = z
-- becomes
(y1 = z AND x1) OR (y2 = z AND x2) OR ...
```

Similarly for the short form:
```sql
(CASE v WHEN u1 THEN y1 WHEN u2 THEN y2 ... END) = z
-- becomes
(y1 = z AND v = u1) OR (y2 = z AND v = u2) OR ...
```

These are crude and can probably be done a lot better.

In the use case mentioned, `yi` is a Const while `z` is a Var.
This should allow the optimizer to remove all terms where `yi = z` is false, leaving only one `xi` term in the first transform or one `v = ui` term in the second, which should be able to use the original indexes.
