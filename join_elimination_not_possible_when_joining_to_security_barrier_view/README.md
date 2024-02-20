# Join elimination not possible when joining to security barrier view

I'm just copying this from my notes. Need to confirm the example is still valid.
-- Bennie

```sql
create table taby as select * from generate_series(1,1e6) idy;
create table tabx as select * from generate_series(1,1e6,1e5) idx;
create unique index on taby using btree (idy);
create view viewy_nobarrier as select * from taby;
create view viewy_barrier with (security_barrier=true) as select * from taby;
explain analyze select * from tabx x left join viewy_nobarrier y on idy = idx; -- fast
explain analyze select * from tabx x left join viewy_barrier y on idy = idx;   -- slow
```
