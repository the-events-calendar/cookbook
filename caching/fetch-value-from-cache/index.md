## Fetching a Value from Cache and Protecting Against Cache Poisoning

Cache poisoning is a situation where a value is incorrectly set in the cache.  
A cached value is considered "incorrect" when it has the wrong type, shape or range.

Cache poisoning can lead to severe consequences such as fatal errors or security vulnerabilities.  
The cause of cache poisoning can be traced back to a bug in the code that sets the value in the cache (e.g., setting a
value that should be a string to an object), a malfunction of the cache (e.g., the cache server crashed or was rebooted
during a write operation), or a malicious actor inserting a value in the cache.  
This last case is the most severe but the least likely to happen.

This concept applies to all types of caches, including memoization and real object-caching.

## Steps

1. Check if the value is set in the cache.

   The value should be stored using [The Events Calendar cache API][1] or the `wp_cache_get` function.

   Checking whether the value is stored in the cache or not is important to avoid a common mistake where a value absent
   from the cache is evaluated as `false` in PHP:

   ```php
   <?php

   // Using wp_cache_get API
   $purchasers_count = wp_cache_get( 'purchasers_count_' . $ticket_id );

   if( ! $purchasers_count ){
       // Fetch the customers from the database or an API ...
   }

   // Using tribe_cache() API
   $purchasers_count = tribe_cache()->get(
      'purchasers_count_' . $ticket_id, 
      Tribe__Cache__Listener::TRIGGER_SAVE_POST
   );

   if( ! $purchasers_count ){
       // Fetch the customers from the database or an API ...
   }
   ```

   What if the count is `0`?  
   Customers should **not** be fetched from the database or the API again, but the code above will fetch them anyway
   since `! $purchasers_count` will evaluate to `true` when `$purchasers_count` is `0`.

   This can be avoided using the `found` parameter of `wp_cache_get`:

   ```php
   <?php

   // wp_cache_get API
   $customers_count = wp_cache_get( 'my_plugin_did_fetch_customers', $group = '', $force = false, $found );

   if( ! $found ){
       // Fetch the customers from the database or an API ...
   }
   
   // tribe_cache() API
   $cached = tribe_cache()->get(
      'purchasers_count_' . $ticket_id,
      Tribe__Cache__Listener::TRIGGER_SAVE_POST,
      null,
      null,
      null,
      $found
   );

   if( ! $found ) ) {
       // Fetch the customers from the database or an API ...
   }
   ```

2. Do not trust the cache: check if the value is of the correct type.

   Once the value is confirmed to be present in the cache, the next step is to check if the value is of the correct
   type.
   Caches, whether they are request-scoped or shared, are loosely typed key-value stores.

   This means no caching system will enforce a type for the value, the code below will not throw any error:

   ```php
   <?php

   // Initially set the value in the cache to a string.
   wp_cache_set( 'my_key', 'a string' );

   // Later, set the value in the cache to an array.
   wp_cache_set( 'my_key', [ 'an', 'array' ] );
   ```

   Since the cache **will not** enforce a type for the value, it is the responsibility of the code that gets the value
   to
   check if the value is of the correct type:

   ```php
   <?php

   use StellarWP\Arrays\Arr;

   // Using the wp_cache_get API
   $cached = wp_cache_get( 'my_stored_array', '', $found );

   if( ! ( $found && Arr::has_shape( $cached, [ 'user_id' => 'is_int', 'level' => 'is_string' ] ) ) ) {
      $array_value = [ /* fetch or build the value ... */ ];
      
      wp_cache_set( 'my_stored_array', $array_value );
   }

   // Using the tribe_cache() API
   $cached = tribe_cache()->get(
      'my_stored_array',
      Tribe__Cache__Listener::TRIGGER_SAVE_POST, 
      null,
      null,
      null,
      $found
   );

   if( ! ( $found && Arr::has_shape( $cached, [ 'user_id' => 'is_int', 'level' => 'is_string' ] ) ) ) {
      $array_value = [ /* fetch or build the value ... */ ];
       
      tribe_cache()->set( 
         'my_stored_array', 
         $array_value, 
         '', Tribe__Cache__Listener::TRIGGER_SAVE_POST
      );   
   }
   ```

   In the example above, the check is against the shape of the array, but it could be any kind of check depending on the
   expected type of the value.

3. Trust the cache: use the value.

   If the value is present in the cache and is of the correct type, use it.

   An over-zealous control of the cached value could invalidate the act of caching something in the first place.

   In the below example, the check costs more than the value itself:

   ```php
   <?php
   
   use StellarWP\Arrays\Arr;

   // Using the tribe_cache() API
   $cached = tribe_cache()->get(
      'my_stored_array',
      Tribe__Cache__Listener::TRIGGER_SAVE_POST,
      null,
      null,
      null,
      $found
   );

   // This is too much: we are firing queries (plural) to check a value meant to save queries!
   $is_user_id = fn( $user_id ): bool => get_user_by( 'ID', $user_id ) instanceof \WP_User;
   $is_level = fn( $level ): bool => in_aray( $level, tribe_get_option( 'customer_levels', [] ) );

   if( $found && Arr::has_shape( $cached, [ 'user_id' => $is_user_id, 'level' => $is_level ] ) ) {
       // Use the value ...
   }
   ```

   In general, check the type, and the shape if applicable, of the value before using it. That is good enough.

### Notes

Protection from cache poisoning is part of defensive programming.  
Static analysis tools, like [PHPStan][2] will point out and require the checks described in this snippet at higher
levels of strictness.

[1]: ./../cache-a-value/index.md

[2]: https://phpstan.org/
