## Finding unbounded queries

An unbounded query is a query that queries the database for a set of values, e.g. IDs or rows, without specifying a
limit.

Unbound queries can be empirically spotted by using a tool like [Query Monitor][1] when they cause a performance issue a
production environment, but they can be hard to spot during development work, as they might not be an issue in
development environments.

For this reason, it is important to have a way to spot unbounded queries during development work in a more stringent
way.

### Steps

1. Decide what page, or request, you want to test for unbounded queries. E.g. a page that lists posts in the backend, or
   a frontend page that lists events, like '/events'.

2. In your development site, enable error logging in the `wp-config.php` file:

    ```php
    define( 'WP_DEBUG', true );
    define( 'WP_DEBUG_LOG', true );
    define( 'WP_DEBUG_DISPLAY', false );
    ```

3. Create a [Throw-away plugin][2] with the following code in the site must-use plugins directory (
   usually `wp-content/mu-plugins`):

   ```php
   <?php
   /**
    * Plugin Name: Find Unbound Queries
    */

   if ( ! defined( 'SAVEQUERIES' ) ) {
       // Start collecting queries information.
       define( 'SAVEQUERIES', true );
   }

   function tec_log_unbound_queries( $query_data, $query, $query_time, $query_callstack ) {
       remove_filter( 'log_query_custom_data', 'tec_log_unbound_queries' );

       // Only log queries that are from Tribe or TEC.
       if ( str_contains( $query_callstack, 'Tribe' ) || str_contains( $query_callstack, 'TEC' ) ) {
           error_log( sprintf( "Unbound query\n\n%s\n\nfrom\n\n%s", $query, $query_callstack ) );
       }

       return $query_data;
   }

   function tec_watch_unbound_queries( $query ) {
       if ( ! is_string( $query ) ) {
           return $query;
       }

       if (
           str_starts_with( $query, 'SELECT' )
           && ! str_contains( $query, 'LIMIT' )
       ) {
           add_filter( 'log_query_custom_data', 'tec_log_unbound_queries', 10, 4 );
       }

       return $query;
   }

   add_filter( 'query', 'tec_watch_unbound_queries', 1000 );
   ```

4. Read the contents of the site debug log, typically `wp-content/debug.log`, and look for entries with the current
   format:

   ```
   [09-Feb-2024 14:39:12 UTC] Unbound query

   SELECT * from wp_posts WHERE post_title LIKE '%hello%';

   from

   require('wp-blog-header.php'), require_once('wp-includes/template-loader.php'), include('/plugins/the-events-calendar/src/views/v2/default-template.php'), get_post_meta, get_metadata, get_metadata_raw, apply_filters('get_post_metadata'), WP_Hook->apply_filters, Tribe__Some__Component->get_all_posts()
   ```
   The plugin will log all queries that _might_ be unbounded: the fact that a query is logged by the plugin does not
   mean that it is unbounded, but it is a good starting point to investigate.

5. To spot unbounded queries, look for queries that are logged by the plugin and that:
    * do not contain a `LIMIT` clause.
    * do not contain a stringent enough `WHERE` clause; e.g. `WHERE post_title LIKE '%hello%'` is not stringent enough,
      as it will return all the posts that contain the word 'hello' in the title. On the same
      note, `WHERE post_type = 'post'` is not stringent enough, as it will return all the posts of type 'post'. A more
      stringent `WHERE` clause would be `WHERE ID in (1,2,3)` that would produce, at most, 3 rows.

6. When in doubt about whether a query is unbounded, and whether it is a problem, answer this question: "How big can the
   number of rows returned by this query if the database contained 1 million rows?". If the answer is "More than 20",
   then the query is unbounded.

7. Once you have collected your results, you can delete the throw-away plugin.

### Notes

Just because a query is returning something that is "small", e.g. "all the post IDs" or "all the event IDs", it does
mean that the query is not unbounded. The size of the single result set is not the issue, the issue is the size of the
result set in the context of the whole database.

[1]: https://wordpress.org/plugins/query-monitor/

[2]: ../../debug/create-throw-away-plugin/index.md
