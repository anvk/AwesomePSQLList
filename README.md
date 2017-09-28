# AwesomePSQLList
My public list of great Postgres Reads as well as queries

## Links

- [work_mem explained OR Is Your Postgres Query Starved for Memory](https://dzone.com/articles/is-your-postgres-query-starved-for-memory)
- [Deep dive into PSQL statistics](https://www.slideshare.net/alexeylesovsky/deep-dive-into-postgresql-statistics-54594192)

## Queries

### Basics

#### Get indexes of tables

```sql
select
    t.relname as table_name,
    i.relname as index_name,
    a.attname as column_name
from
    pg_class t,
    pg_class i,
    pg_index ix,
    pg_attribute a
where
    t.oid = ix.indrelid
    and i.oid = ix.indexrelid
    and a.attrelid = t.oid
    and a.attnum = ANY(ix.indkey)
    and t.relkind = 'r'
    and t.relname in ('table1', 'table2')
order by
    t.relname,
    i.relname;
```

### Right this second

#### Show running queries

```sql
SELECT pid, age(query_start, clock_timestamp()), usename, query  FROM pg_stat_activity WHERE query != '<IDLE>' AND query NOT ILIKE '%pg_stat_activity%' ORDER BY query_start desc;
```

#### Queries which are running for more than 2 minutes

```sql
SELECT now() - query_start as "runtime", usename, datname, waiting, state, query FROM pg_stat_activity WHERE now() - query_start > '2 minutes'::interval ORDER BY runtime DESC;
```

#### Queries which are running for more than 9 seconds

```sql
SELECT now() - query_start as "runtime", usename, datname, waiting, state, query FROM pg_stat_activity WHERE now() - query_start > '9 seconds'::interval ORDER BY runtime DESC;
```

#### Kill running query

```sql
SELECT pg_cancel_backend(procpid);
```

#### Kill idle query

```sql
SELECT pg_terminate_backend(procpid);
```

#### Vacuum Command

```sql
VACUUM (VERBOSE, ANALYZE);
```

### Data Integrity

#### Cache Hit Ratio

```sql
select sum(blks_hit)*100/sum(blks_hit+blks_read) as hit_ratio from pg_stat_database;
```
(perfectly )hit_ration should be > 90%

#### Anomalies

```sql
select datname, (xact_commit*100)/(xact_commit+xact_rollback) as c_commit_ratio, (xact_rollback*100)/(xact_commit+xact_rollback) as c_rollback_ratio, deadlocks, conflicts, temp_files, pg_size_pretty(temp_bytes) from pg_stat_database;
```

c_commit_ratio should be > 95%
c_rollback_ratio should be < 5%
deadlocks should be close to 0
conflicts should be close to 0
temp_files and temp_bytes  watch for them

#### Table Sizes

```sql
select relname, pg_size_pretty(pg_total_relation_size(relname::regclass)) as full_size, pg_size_pretty(pg_relation_size(relname::regclass)) as table_size, pg_size_pretty(pg_total_relation_size(relname::regclass) - pg_relation_size(relname::regclass)) as index_size from pg_stat_user_tables order by pg_total_relation_size(relname::regclass) desc limit 10;
```

#### Another Table Sizes Query

```sql
SELECT nspname || '.' || relname AS "relation", pg_size_pretty(pg_total_relation_size(C.oid)) AS "total_size" FROM pg_class C LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace) WHERE nspname NOT IN ('pg_catalog', 'information_schema') AND C.relkind <> 'i' AND nspname !~ '^pg_toast' ORDER BY pg_total_relation_size(C.oid) DESC;
```

#### Database Sizes

```sql
select datname, pg_size_pretty(pg_database_size(datname)) from pg_database order by pg_database_size(datname);
```

#### Unused Indexes

```sql
select * from pg_stat_all_indexes where idx_scan = 0;
```
idx_scan should not be = 0

#### Write Activity(index usage)

```sql
select s.relname, pg_size_pretty(pg_relation_size(relid)), coalesce(n_tup_ins,0) + 2 * coalesce(n_tup_upd,0) - coalesce(n_tup_hot_upd,0) + coalesce(n_tup_del,0) AS total_writes, (coalesce(n_tup_hot_upd,0)::float * 100 / (case when n_tup_upd > 0 then n_tup_upd else 1 end)::float)::numeric(10,2) AS hot_rate, (select v[1] FROM regexp_matches(reloptions::text,E'fillfactor=(d+)') as r(v) limit 1) AS fillfactor from pg_stat_all_tables s join pg_class c ON c.oid=relid order by total_writes desc limit 50;
```
hot_rate should be close to 100

#### Does table need an index?
```sql
SELECT relname, seq_scan-idx_scan AS too_much_seq, CASE WHEN seq_scan-idx_scan>0 THEN 'Missing Index?' ELSE 'OK' END, pg_relation_size(relname::regclass) AS rel_size, seq_scan, idx_scan FROM pg_stat_all_tables WHERE schemaname='public' AND pg_relation_size(relname::regclass)>80000 ORDER BY too_much_seq DESC;
```

#### Index % usage

```sql
SELECT relname, 100 * idx_scan / (seq_scan + idx_scan) percent_of_times_index_used, n_live_tup rows_in_table FROM pg_stat_user_tables ORDER BY n_live_tup DESC;
```

#### How many indexes are in cache

```sql
SELECT sum(idx_blks_read) as idx_read, sum(idx_blks_hit) as idx_hit, (sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit) as ratio FROM pg_statio_user_indexes;
```

#### Dirty Pages

```sql
select buffers_clean, maxwritten_clean, buffers_backend_fsync from pg_stat_bgwriter;
```
maxwritten_clean and buffers_backend_fsyn better be = 0

#### Sequential Scans

```sql
select relname, pg_size_pretty(pg_relation_size(relname::regclass)) as size, seq_scan, seq_tup_read, seq_scan / seq_tup_read as seq_tup_avg from pg_stat_user_tables where seq_tup_read > 0 order by 3,4 desc limit 5;
```
seq_tup_avg should be < 1000

#### Checkpoints

```sql
select 'bad' as checkpoints from pg_stat_bgwriter where checkpoints_req > checkpoints_timed;
```

### Activity

#### Most CPU intensive queries (PGSQL v9.4)

```sql
SELECT substring(query, 1, 50) AS short_query, round(total_time::numeric, 2) AS total_time, calls, rows, round(total_time::numeric / calls, 2) AS avg_time, round((100 * total_time / sum(total_time::numeric) OVER ())::numeric, 2) AS percentage_cpu FROM pg_stat_statements ORDER BY total_time DESC LIMIT 20;
```

#### Most time consuming queries (PGSQL v9.4)

```sql
SELECT substring(query, 1, 100) AS short_query, round(total_time::numeric, 2) AS total_time, calls, rows, round(total_time::numeric / calls, 2) AS avg_time, round((100 * total_time / sum(total_time::numeric) OVER ())::numeric, 2) AS percentage_cpu FROM pg_stat_statements ORDER BY avg_time DESC LIMIT 20;
```

#### Maximum transaction age

```sql
select client_addr, usename, datname, clock_timestamp() - xact_start as xact_age, clock_timestamp() - query_start as query_age, query from pg_stat_activity order by xact_start, query_start;
```
Long-running transactions are bad because they prevent Postgres from vacuuming old data. This causes database bloat and, in extreme circumstances, shutdown due to transaction ID (xid) wraparound. Transactions should be kept as short as possible, ideally less than a minute.

#### Bad xacts

```sql
select * from pg_stat_activity where state in ('idle in transaction', 'idle in transaction (aborted)');
```

#### Waiting Clients

```sql
select * from pg_stat_activity where waiting;
```

#### Waiting Connections for a lock

```sql
SELECT count(distinct pid) FROM pg_locks WHERE granted = false;
```

#### Connections

```sql
select client_addr, usename, datname, count(*) from pg_stat_activity group by 1,2,3 order by 4 desc;
```

#### User Connections Ratio

```sql
select count(*)*100/(select current_setting('max_connections')::int) from pg_stat_activity;
```

#### Average Statement Exec Time

```sql
select (sum(total_time) / sum(calls))::numeric(6,3) from pg_stat_statements;
```

#### Most writing (to shared_buffers) queries

```sql
select query, shared_blks_dirtied from pg_stat_statements where shared_blks_dirtied > 0 order by 2 desc;
```

#### Block Read Time

```sql
select * from pg_stat_statements where blk_read_time <> 0 order by blk_read_time desc;
```

### Vacuuming

#### Last Vacuum and Analyze time

```sql
select relname,last_vacuum, last_autovacuum, last_analyze, last_autoanalyze from pg_stat_user_tables;
```

#### Total number of dead tuples need to be vacuumed per table

```sql
select n_dead_tup, schemaname, relname from pg_stat_all_tables;
```

#### Total number of dead tuples need to be vacuumed in DB

```sql
select sum(n_dead_tup) from pg_stat_all_tables;
```

