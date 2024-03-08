## Understand if a query is slow or fast

This recipe will help you understand if a query is slow or fast.  

A first indication that a query is slow might be the indication, from a plugin like [Query Monitor][1], that the query took too long.  
By default, this means the query took longer than 0.05 seconds to run; that would be a 1/20th of a second.  

Sometimes, though, a query might be slow even if it takes less than 0.05 seconds to run.  

As an example, an inefficient query that runs on an empty database will run fast, but as the database grows, the query will become slower and slower.

Understanding if a query is slow or fast is, then not just a matter of detecting slow queries, but to look for them **before** they become slow.  

### Steps

1. Use [Query Monitor][1] to spot slow queries.

   Install and activate the plugin, and navigate to the page where you want to test the query.  
   Reload the page and look at the "Queries" tab in the Query Monitor panel: if the query took more than 0.05 seconds to run, it will be highlighted in red.

2. Is the query running during a READ or WRITE request?

   There are 2 main types of requests to the site: READ and WRITE.  
   A "READ" request is one where information is exclusively read from the database, likely originating from an HTTP verb like `GET`.
   A "WRITE" request is on where information is read from, and written to, the database. These requests likely originated from `POST`, `PUT`, `DELETE` HTTP verbs.  

   If the slow query is happening only in the context of "WRITE" requests, it _could_ be worth investigating, but your first objective should be to optimize queries happening in the context of "READ" requests.  

2. If the query you're looking is not reported as slow by Query Monitor, but you suspect it might be slow, populate your development database with a large amount of data and run the query again.
   
   If, for example, the query fetches Events from the `wp_posts` table (`post_type = 'tribe_events'`), fill the `wp_posts` table with a large amount of data and run the query again until the query becomes slow. 
   If the query becomes slow at very high values, e.g. 1 million Events, then it's likely not slow; you can stop here.

3. Using Query Monitor, copy the query SQL code of the query.

   You should end with code like this one:  

   ```sql
   SELECT * from wp_posts books, wp_postmeta blurbs
   WHERE books.post_type = 'book'
   AND blurbs.post_id = 23
   AND blurbs.meta_key = '_blurb';
   ```

4. Is the query fetching too many fields?

   Whether the query is built using the ORM (e.g. `tribe_events()` or `tribe_tickets()`) or by using [the `stellarwp\db` library][2], the query should only fetch, from the database the minimum required set of fields.  

   WordPress, by means of the `WP_Query` object, and The Events Calendar ORMs, are optimized in this sense, but the query might be correct, just used wrongly.  
  
   To understand if the query is fetching too many fields, find where the query is being built. Again here Query Monitor is your friend, the `Caller` section of the report will provide you with a trace to point you to the origin of the query.  

   Imagine you find the code originating a slow query is the one below.  
   Creating 10,000 upcoming Events was slowing your development site down to a crawl:  

   ```php
   <?php

   $events = tribe_events()->where( 'starts_after', 'now' )->all();

   if( count( $events ) ) {
       printf( "There are upcoming events\n" );
   } else {
       printf( "There are no upcoming events\n" );
   }
   ```

   The query, built by the `tribe_events()` ORM, is pulling **all** the post fields for an Event (and the Event meta) to count the Events.  
   This query should be rewritten to pull out a single number from the database, the number of upcoming Events, like this:

   ```php
   $count = tribe_events()->where( 'starts_after', 'now' )->count();

   if( $count > 1 ) {
       printf( "There are upcoming events\n" );
   } else {
       printf( "There are no upcoming events\n" );
   }
   ```

   If refactoring the query, or the query caller, to fetch less fields, slowed the slow query issue, you're done.  

5. Is the query unbounded?  

   Unbounded queries are bad for all sorts of reasons, read more in the [find unbounded queries recipe][3] recipe.

   If the query is unbounded, refactor it to a bounded query, following the steps in the [refactor unbounded query][4] recipe.
   
   Fetching the same amount of information in chunks, will make the query faster.

   If refactoring the query to be bounded, slowed the slow query issue, you're done.

6. Is the query correctly written?
   
   At step 3, you copied the query SQL code, now it's time to `EXPLAIN` it.  
   
   Using the information from the `EXPLAIN` command, you can understand if the query is correctly written, and if it's using the available indexes.

   This recipe cannot, for the sake of completeness and brevity, guide you through all the information provided by the output of the `EXPLAIN` command, you can find a complete guide on the [MySQL documentation][5].

   Read more about some quick take-aways about indexes in the [Notes](#notes) section.
   
### Notes

Here are some indications about how to start thinking about indexes in association with fast and slow queries:

* **Sub-queries are not inherently slow**
  But they can be slow if not written correctly.  
  If you cannot write a `JOIN` query, and you need to use a sub-query, make sure the sub-query is correctly written: just debug it like you would for a query.  
  
* **Know your indexes.**  
  Use [the `SHOW INDEXES FROM <table>` command][6] in MySQL to understand which indexes are available for a table.
  As an example, the `wp_posts` table (`SHOW INDEXES FROM wp_posts`) has an index on the `ID` and `post_name` fields, as expected.  

  The following query will hit the (primary) `ID` index and will be, intuitively, very fast:

  ```
  mysql> EXPLAIN SELECT * FROM wp_posts WHERE ID = 23;
  +----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
  | id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
  +----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
  |  1 | SIMPLE      | wp_posts | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL  |
  +----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
  ```

  But the table has also a multiple field index on `post_type, post_status, post_date, ID` called `type_status_date`.  
  Knowing this we can write another very fast query:

  ```
  mysql> EXPLAIN SELECT * FROM wp_posts WHERE post_type = 'tribe_events' and post_status = 'private';
  +----+-------------+----------+------------+------+------------------+------------------+---------+-------------+------+----------+-------+
  | id | select_type | table    | partitions | type | possible_keys    | key              | key_len | ref         | rows | filtered | Extra |
  +----+-------------+----------+------------+------+------------------+------------------+---------+-------------+------+----------+-------+
  |  1 | SIMPLE      | wp_posts | NULL       | ref  | type_status_date | type_status_date | 164     | const,const |    1 |   100.00 | NULL  |
  +----+-------------+----------+------------+------+------------------+------------------+---------+-------------+------+----------+-------+
  ```

  Using `SHOW INDEXEX FROM wp_postmeta` will show there is a `meta_key` index (**not** a `meta_value` one), this can be leveraged to write fast `JOIN` queries:

  ```
  mysql> EXPLAIN SELECT * FROM wp_posts events JOIN wp_postmeta start_date ON events.ID = start_date.post_id AND start_date.meta_key = '_EventStartDate' WHERE start_date.meta_value > '2022-10-12 08:00:00';
  +----+-------------+------------+------------+--------+------------------+----------+---------+------------------------------+------+----------+-------------+
  | id | select_type | table      | partitions | type   | possible_keys    | key      | key_len | ref                          | rows | filtered | Extra       |
  +----+-------------+------------+------------+--------+------------------+----------+---------+------------------------------+------+----------+-------------+
  |  1 | SIMPLE      | start_date | NULL       | ref    | post_id,meta_key | meta_key | 767     | const                        |    7 |    33.33 | Using where |
  |  1 | SIMPLE      | events     | NULL       | eq_ref | PRIMARY          | PRIMARY  | 8       | wordpress.start_date.post_id |    1 |   100.00 | NULL        |
  +----+-------------+------------+------------+--------+------------------+----------+---------+------------------------------+------+----------+-------------+
  
  mysql> EXPLAIN SELECT * FROM wp_posts events JOIN wp_postmeta start_date ON events.ID = start_date.post_id AND start_date.meta_key LIKE '_EventStartDate' WHERE start_date.meta_value > '2022-10-12 08:00:00';
  +----+-------------+------------+------------+--------+---------------+---------+---------+------------------------------+------+----------+-------------+
  | id | select_type | table      | partitions | type   | possible_keys | key     | key_len | ref                          | rows | filtered | Extra       |
  +----+-------------+------------+------------+--------+---------------+---------+---------+------------------------------+------+----------+-------------+
  |  1 | SIMPLE      | start_date | NULL       | ALL    | post_id       | NULL    | NULL    | NULL                         | 1334 |     3.70 | Using where |
  |  1 | SIMPLE      | events     | NULL       | eq_ref | PRIMARY       | PRIMARY | 8       | wordpress.start_date.post_id |    1 |   100.00 | NULL        |
  +----+-------------+------------+------------+--------+---------------+---------+---------+------------------------------+------+----------+-------------+
  ```

  Beware of **how** you leverage an index: the queries will return the same values, but the second one, using `start_date.meta_key LIKE '_EventStartDate'` in place of `start_date.meta_key LIKE '_EventStartDate'`, will **not** use the index on `meta_key`. Only some comparision operators (e.g. `=` and not `LIKE`) will fully leverage an index.  

* **Table `JOINS` are not inherently slow.**
  
  "Too many JOINs" **is** a problem, but take the time to `EXPLAIN` why a `JOIN` is slow.
    
  As a first measure, the `JOIN`s should hit an index:

  ```
  mysql> EXPLAIN SELECT * FROM wp_posts p JOIN wp_postmeta keyword ON keyword.post_id = p.ID AND keyword.meta_key LIKE '_%_keyword' AND keyword.meta_value = 'lorem';
  +----+-------------+---------+------------+--------+---------------+---------+---------+---------------------------+------+----------+-------------+
  | id | select_type | table   | partitions | type   | possible_keys | key     | key_len | ref                       | rows | filtered | Extra       |
  +----+-------------+---------+------------+--------+---------------+---------+---------+---------------------------+------+----------+-------------+
  |  1 | SIMPLE      | keyword | NULL       | ALL    | post_id       | NULL    | NULL    | NULL                      | 1334 |     1.11 | Using where |
  |  1 | SIMPLE      | p       | NULL       | eq_ref | PRIMARY       | PRIMARY | 8       | wordpress.keyword.post_id |    1 |   100.00 | NULL        |
  +----+-------------+---------+------------+--------+---------------+---------+---------+---------------------------+------+----------+-------------+
  ```

  This will be a slow query: the `JOIN` does not happen on an index (look at the `keyword` table row), MySQL estimates it will have to look up 1334 rows to keep 1.11% of them.  
  If we know the meta key to look for is either `_cornerston_keyword` or `_additional_keyword` the same query can be rewritten to use the same `JOIN`s hitting the `meta_key` index on the `wp_postmeta` table:

  ```
  mysql> EXPLAIN SELECT * FROM wp_posts p JOIN wp_postmeta keyword ON keyword.post_id = p.ID AND keyword.meta_key = '_cornerstone_keyword' OR keyword.meta_key = '_additional_keyword' AND keyword.meta_key = 'lorem';
  +----+-------------+---------+------------+--------+------------------+----------+---------+---------------------------+------+----------+-------------+
  | id | select_type | table   | partitions | type   | possible_keys    | key      | key_len | ref                       | rows | filtered | Extra       |
  +----+-------------+---------+------------+--------+------------------+----------+---------+---------------------------+------+----------+-------------+
  |  1 | SIMPLE      | keyword | NULL       | ref    | post_id,meta_key | meta_key | 767     | const                     |    1 |   100.00 | Using where |
  |  1 | SIMPLE      | p       | NULL       | eq_ref | PRIMARY          | PRIMARY  | 8       | wordpress.keyword.post_id |    1 |   100.00 | NULL        |
  +----+-------------+---------+------------+--------+------------------+----------+---------+---------------------------+------+----------+-------------+
  ```

  This query will provide the same results but the `JOIN` is hsirring the `meta_key` index (look for `key`).

  As a rule of thumb: unroll `meta_keys`. If you know the `meta_key` will be in a set of values, do not use `LIKE` or `IN`, use a set of `=` comparisons.

[1]: https://wordpress.org/plugins/query-monitor/
[2]: https://github.com/stellarwp/db
[3]: ../find-unbounded-queries/index.md
[4]: ../refactor-unbounded-query/index.md
[5]: https://dev.mysql.com/doc/refman/8.0/en/explain.html
[6]: https://dev.mysql.com/doc/refman/8.0/en/show-index.html
