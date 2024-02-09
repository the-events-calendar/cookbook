## Refactor an unbounded query to a bounded query

An unbounded query is a query that queries the database for a set of values, e.g. IDs or rows, without specifying a
limit.

This leads to a performance issue, as the database will return all the rows that match the query, which can be a large
number of rows.  
Those rows will be moved back to the PHP process, which can lead to memory issues as that data of unknown size is being
stored in RAM.

Unbound queries cannot be always spotted during development work, as they might not be an issue in development
environments, but they can be a big issue in production environments.  
A tool like [Query Monitor][1] can help to spot unbounded queries when they happen in development environments, but a
development environment might not have the same amount of data as a production environment and the issue might not be
spotted.

If you're unsure if a query is unbounded, look at the [Finding unbounded queries][2] recipe.

The following steps assume you've found the unbound query origin in the code, and you want to refactor it to a
non-unbound query.

### Steps

1. If the query is generated by a TEC ORM, like `tribe_events()`, `tribe_tickets()`, or `tribe_attendees()` go
   to [Refactor an unbounded TEC ORM query][3].

2. If the query is generated by `WP_Query` and is for post types managed by The Events Calendar
   suite, [refactor the query to use the ORM][3].

3. If the query is a direct query to the database done using the global `$wpdb` object, refactor the query to use
   the `StellarWP\DB` class:

    ```diff
    $query = "SELECT * FROM {$wpdb->posts} WHERE post_type = 'tribe_events' AND post_status = 'private'"; 
    - global $wpdb;
    - $private_events = $wpdb->get_results( $query );
    + $private_events = \TEC\Common\StellarWP\DB\DB::generate_results( $query );
   
    foreach( $private_events as $private_event ) {
        // Do something with the private event.
    }
    ```

   With columns:

    ```diff
    $query = "SELECT ID FROM {$wpdb->posts} WHERE post_type = 'tribe_events' AND post_status = 'private'";
    - global $wpdb;
    - $private_events_ids = $wpdb->get_col( $query );
    + $private_events_ids = \TEC\Common\StellarWP\DB\DB::generate_col( $query );
   
    foreach( $private_events_ids as $private_event_id ) {
        // Do something with the private event ID.
    }
    ```

4. If the query is a direct query to the database done using the `\TEC\Common\StellarWP\DB\DB` class, refactor the query
   to use the `generate_` methods in place of the `get_` ones:

    ```dif
    $query = "SELECT * FROM {$wpdb->posts} WHERE post_type = 'tribe_events' AND post_status = 'private'";
    - $private_events = \TEC\Common\StellarWP\DB\DB::get_results( $query );
    + $private_events = \TEC\Common\StellarWP\DB\DB::generate_results( $query, 10 );
   
    foreach( $private_events as $private_event ) {
        // Do something with the private event.
    }
    ```

   With columns:

    ```diff
    $query = "SELECT ID FROM {$wpdb->posts} WHERE post_type = 'tribe_events' AND post_status = 'private'";
    - $private_events_ids = \TEC\Common\StellarWP\DB\DB::get_col( $query );
    + $private_events_ids = \TEC\Common\StellarWP\DB\DB::generate_col( $query, 50 );
   
    foreach( $private_events_ids as $private_event_id ) {
        // Do something with the private event ID.
    }
    ```

### Notes

[1]: https://wordpress.org/plugins/query-monitor/

[2]: ../finding-unbounded-queries/index.md

[3]:  ../refactor-unbounded-tec-orm-query/index.md

[4]: https://www.php.net/manual/en/language.generators.overview.php