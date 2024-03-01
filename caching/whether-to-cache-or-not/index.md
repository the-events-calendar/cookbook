## Deciding whether a value should be cached or not

Deciding whether a value should be cached or not is not something to be taken lightly.

> There are only two hard things in Computer Science: cache invalidation and naming things. -- Phil Karlton

The burden of caching is that of deciding **when the cache should be invalidated** or, in other words, when the value
stored in the cache should be considered stale and evicted or ignored.

This recipe will guide you through the process of deciding whether a value should be cached ,memoized or computed in the
context of WordPress and The Events Calendar suite of plugins.

Some definitions to keep in mind when reading this recipe:

* **Caching** is the process of storing a value in a cache, like Redis, Memcached, or the WordPress object cache
    * This value will be stored for a certain amount of time, and will be evicted after that time
    * This value will be shared across all the requests to the site until it's evicted

* **Memoization** is the process of storing a value in memory, i.e. in the PHP process handling the current request.
    * This value will be stored for the duration of the request
    * This value will be shared across all the functions and methods in the current request

* **Computed** is the process of calculating the value in real time, when not found in the cache or memory
    * This value will be calculated on each request
    * This value will not be shared across requests
    * This value will not be shared across all the functions and methods in the current request

Cache systems like Redis and Memcached are **another database**.  
They are a key-value store, where the key is the cache key and the value is the cached value.  
They are very fast databases, but they are still databases that need to be queried and written to through sockets or,
more commonly, TCP connections like the MySQL database.  
You can think of them as a MySQL database with a single table with two columns: `key` and `value`.

A WordPress site will have two main type of requests: `READ` and `WRITE`.

A `READ` request is one where a visitor or user is looking at the site content in one format or another.  
Examples of `READ` requests are:

* A visitor looking at the site's homepage
* A site administrator looking at the site's dashboard or list of posts
* A REST API request to fetch a list of posts

A `WRITE` request is one where the site content is updated, i.e. the state of the site changes in the database.
Examples of `WRITE` requests are:

* A site administrator updating a post
* A visitor submitting a comment
* A REST API request to create a new post

A rough estimate is that 99% of the requests to a WordPress site are `READ` requests, and 1% are `WRITE` requests.

### Steps

1. Is the value required **exclusively** in a `WRITE` request?

   If yes and the value will change in the context of the `WRITE` request, do not cache it.

   If yes and the value will not change in the context of the `WRITE` request, you can memoize it.

   If no, the value is required in both `READ` and `WRITE` requests, go to the next step.

2. Could the value change in the context of the request?

   E.g. the HTML produced from a template engine might change depending on the data to render, the filter and actions
   applied while rendering it.
   The template is the same, but the output HTML is different.

   If the value could change in the context of the request, do not cache it.

   If the value will not change in the context of the request, go to the next step.

3. How costly is the value to calculate?

   E.g. getting the number of posts of a specific post type, of a specific post status, per user, is one query:

    ```php
    <?php
   
    use StellarWP\DB\DB;
   
    global $wpdb;

    $query = "SELECT COUNT(*) FROM %i WHERE post_type = %s AND post_status = %s AND post_author = %d";
    $count = DB::get_var( 
        DB::prepare( $query, $wpdb->posts, 'tribe_events', 'private', $user_id ) 
    );
    ```

   This is a fast query on a single table hitting indexed columns.

   Storing and fetching the value from the cache would likely be comparable: one query to a database.

   If the value is computed in PHP, how many operations are required to compute the value?

   If the value is fetched from an external API, go to the next step.

   **If the value can be calculated in a single query, or a few operations, don't cache it, memoize it.**

   Take some time to think if you could rewrite the value computation to be faster, e.g. in a single query, or fewer
   operations.

   If the value is slow to compute, go to the next step.

4. Will the value change across requests due to time?

   The value might change depending on time.

   E.g. a list of upcoming Events IDs would change as time passes.
   Or data from an external API might change as time passes.

   If time is the **only** factor that would make the value stale, you can cache for half the minimum change time.

   E.g. Event times are all set in multiples of 15'. You can cache the list of upcoming Events for 7'30". I.e. double
   the frequency of change, half the time.

   E.g. if an external API changes every hour, you should cache the data for 30'.

   If time is not the **only** factor that would make the value stale, go to the next step.

5. What watchers should be set up to invalidate the value? When would those watchers run?

   If the value could change due to other factors, take the time to list all the possible source of change that would
   make the value stale.

   E.g. a list of upcoming Events could change depending not only on time, but also on the access level of the user
   looking at them or the linked posts (venues, organizers, tickets, etc.).
   The factors that would make the list of upcoming Events stale are:

    * Time
    * Events are created, updated or deleted
    * Linked posts are created, updated or deleted
    * User access level changes (e.g. visitors cannot see the Event organizers, but logged in users can)

6. For each source of change, decide whether it should be:

    * a key salt (something use to build the cache key)
    * an expiration time
    * something to watch for changes

   E.g. looking at the list above:
    * Time - there is nothing to watch if not time itself, this is an expiration time (e.g. 7'30")
    * Events are created, updated or deleted - this is something to watch
    * Linked posts are created, updated or deleted - this is something to watch
    * User access level changes - this is a key salt, i.e. the user ID or user role

   If all sources of change are either expiration of salts, cache the value.

7. For each watcher, write down how you would watch it for changes to trigger a cache invalidation.

   A "watcher" is a component that, usually by means of a hook, will trigger a cache invalidation when the source of
   change happens.  
   A watcher _might_ also refresh the cache, but that is not always the case.

   E.g. looking at the list above (request context in parentheses):

    * Events are created, updated or deleted: hook on the `save_post_tribe_events` and `deleted_post` actions. (`WRITE`)
    * Linked posts are created, updated or deleted: hook on the `save_post_{$post_type}` and `deleted_post` actions for
      each linked post type. (`WRITE`)

   If all watchers would run in the context of `WRITE` request, cache the value.

   If one or more watchers would run in the context of `READ` requests, you're doing it wrong.  
   Take the time to think about how to run watchers only in the context of `WRITE` requests.  
   If required, re-architect or refactor the code to run watchers only in the context of `WRITE` requests.

[1]: ../../database-queries/understand-if-a-query-is-slow-or-fast/index.md
