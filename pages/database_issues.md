---
title: database issues
---

## Research

I reviewed all the material over the last few weeks and info from Scout, deploy times, code changes, and the logs. Listed below are the recommendations I think we should do for this round of site improvements.


## database issues

- [ ] db upgrade
- [ ] touching users too often
- [ ] brief webhooks
- [ ] reforge webhooks
- [ ] sidebar overload
- [ ] slow Event pages
- [ ] userfunnelstate
- [ ] cmssection i/o
- [ ] bg jobs
- [ ] update_video_progress
- [ ] memory leaks on web


### db upgrade
A potential low effort solution that might help is to just up the RAM in the db. We are at 4gb right now and the next tier is 8gb

### seperate reads and writes
[reads and writes](https://blog.saeloun.com/2019/12/10/rails-block-writes-to-database-connection-while-prevent-writes.html)
Another solution is worth exploring is adding read DBs. Will look into this further before moving forward.

### Touch
The number of requests we've been logging has [ballooned](https://i.imgur.com/ceMoAXw.png). Caches break so often that it's useless to the UX.

1. Remove the sidebar cache
2. Remove user touch events for user_program; program_progress; activity
3. Optimize sidebar
  1. Lazy load
  2. Cache parts of it like industries; group names; programs
4. Think about caching on a user basis that does not involve using updated_at

#### Notes from the database

During spikes, run these queries

Indexes in cache and the rates they are hit. A healthy db sees 99% hits
```sql
SELECT sum(heap_blks_read) as heap_read, sum(heap_blks_hit)  as heap_hit, (sum(heap_blks_hit) - sum(heap_blks_read)) / sum(heap_blks_hit) as ratio
FROM pg_statio_user_tables;
```


What tables are being locked?
[here is an example of our users table being locked](https://i.imgur.com/4qWyg5f.png)
```sql
select nspname,relname,l.* from pg_locks l join pg_class c on 
 (relation=c.oid) join pg_namespace nsp on (c.relnamespace=nsp.oid) where 
  pid in (select pid from pg_stat_activity where 
  datname=current_database() and query!=current_query());
```

another way
```bash
{23:28}~/code/reforge/reforge:master ✗ ➭ heroku pg:locks -r production
 pid | relname | transactionid | granted | query_snippet | age
-----+---------+---------------+---------+---------------+-----
(0 rows)
```

Logs indicate no deadlocks.
[further reading on checkpoints and writing to disc](https://postgreshelp.com/postgresql-checkpoint/)


Checking bloat
```bash
heroku pg:bloat -r production
```
Our bloat is low, btw. Looking for bloat factor over 10

Checking vacuum stats
```bash
heroku pg:vacuum_stats -r production
```
Our stats look fine.

[further reading on database garbage collection](https://devcenter.heroku.com/articles/managing-vacuum-on-heroku-postgres)

#### Memory leak from a web task

At exactly 8:30, memory increased and was permanent. This [image shows by how much](https://i.imgur.com/1NjCr2r.png). What's interesting is that this is from a web job, not a bg job, indicating that this was driven by a user.
My assumption is that it's the event import script: import_attendance, ran by eric near that time


#### heavy bg jobs

SearchDocumentJob [link](https://scoutapm.com/apps/157583/workers/Sm9iL1NlYXJjaERvY3VtZW50Sm9i?section=performance#QWN0aXZlUmVjb3JkL0Ntc1NlY3Rpb24vZmluZA==)
image: [CmsSection](https://i.imgur.com/9DdM5xy.png)
Not a huge win, but moving this to hourly or to a third party for search would be ideal. Our search is not heavily used yet

- [ ] *ActivityJob

#### EventsController is bad

- Show has nearly 1000 queries (over two minutes average load time)
- Links is at nearly 17 second average
- AdminEventsIndex is also nearly 1000 queries


#### our followers fall behind

[image](https://i.imgur.com/v8Zdo9r.png)

it's okay for it to fall behind during backups as indicated in checkpoints but as the database grows larger, the backups take longer to complete:

```bash
2020-09-24T19:49:50.000000+00:00 app[postgres.135]: [PINK] [18751-1]  sql_error_code = 00000 LOG:  checkpoint complete: wrote 3005 buffers (3.0%); 0 WAL file(s) added, 0 removed, 9 recycled; write=301.200 s, sync=0.004 s, total=301.224 s; sync files=93, longest=0.002 s, average=0.000 s; distance=130790 kB, estimate=343227 kB
2020-09-24T19:49:51.000000+00:00 app[postgres.4456]: [PINK] [21-1]  sql_error_code = 00000 LOG:  duration: 301365.830 ms  statement: SELECT json_build_object('at', 'pg_start_backup', 'lsn', pg_start_backup('8dec463b_8c13_452c_a9e4_0d51c4b0d50f', false, false));
```
Still unclear on any actions to take here.


#### leads should be turned off

This comes from brief webhooks and is hammered every time we send out a marketing email. Potentially 100,000s of db hits
```sql
SELECT $1 AS one
FROM "leads"
WHERE LOWER("leads"."email") = LOWER($2)
        AND "leads"."id" != $3 LIMIT $4
```


#### CmsSection.find has an I/O problem

It's got a lot of bloat inside of it due to the multiple text fields.
Some options to consider
1. Cache them sitewide. they don't update that often
2. Use `.select` to just get the columns you want when quering for TOC and dashboard items
3. Moving the content itself into a Content table and linking everything that way instead
