## Caching a value

Caching a value is a pretty straightforward process, but there are additional consideration when it comes to doing it in
the context of The Events Calendar suite of plugins.

### Steps

1. Decide whether the value should be cached (or memoized) or not.

   If you've not done so already, read the [recipe about deciding whether to cache a value or not][1] to guide your
   choice.

   If you think the value should be memoized move to step `2`.

   If you think the value should be cached move to step `3`.

2. Decide what are the cache key salts (components).

   Common salts for the cache key are the `__FUNCTION__`, `__CLASS__` and `__METHOD__` constants.  
   Do not use them if you need to reference the value in the context of other functions, classes or methods.

   Other common salts are:

    * current user ID
    * current user role
    * output format (e.g. 'html' or `json`)
    * input arguments (e.g. the function parameters)

   Some or all of them could apply to the value you're going ot cache or memoize.

   Do not use the result of functions like `date`, `time` and `microtime` as salts.  
   If you think you should use them, you should neither cache nor memoize the value.

3. Memoize the value

   The Events Calendar suite of plugins uses an abstraction layer over WordPress cache functions to handle cache groups
   and invalidation triggers transparently.
   The API for this abstraction layer is the `tribe_cache()` function that will manage, under the hood, the singleton
   instance of teh `Tribe__Cache` class.

   Memoization will store the value in a unique, non-persistent across requests, cache group.

   ```php
   <?php

   function tec_fetch_user_visible_events(int $count, int $from, string $format): string {
       $cache = tribe_cache();
       $cache_key = __FUNCTION__ . $count . $from . $format;

       if( !isset( $cache[ $cache_key ] ) ) {
           return $cache[ $cache_key ];
       }
        
       $events = tribe_events()->offset( $from )->per_page( $count )->get_ids() );

       $cache[ $cache_key ] = $events;

       return $events;
   }
   ```

   Note that, since memoization will not survive across requests, some cache key components can be skipped.  
   E.g. the current user ID or role, thus access level, would influence what Events are visible to them.  
   The user ID is not used as salt of the cache key since it's likely not going to change in the context of the request.

   You're done.

4. Decide when the value should be invalidated

   The same facility used for memoization, `tribe_cache()` is used for caching.

   The cache expiration can be set in the following ways:

   a. until an amount of seconds has passed  
   b. until a post of a type controlled by The Events Calendar suite (Events, Venues, Organizers, Tickets) is created,
   updated or deleted
   * optionally, you can also specify an amount in seconds: whatever condition will occurr first will invalidate the
   cached value
   c. until the main The Events Calendar option (`tribe_events_calendar_options`) is updated  
   * optionally, you can also specify an amount in seconds: whatever condition will occurr first will invalidate the
   cached value
   d. until rewrite rules are regenerated
   * optionally, you can also specify an amount in seconds: whatever condition will occurr first will invalidate the
   cached value

   Depending on the nature of the value you're caching, pick the best option and go to the corresponding step.

5. Cache the value until an amount of seconds has passed

   The same facility used for memoization, `tribe_cache()` is used for caching.

   Caching will store the value in a single group that will be valid across all requests until invalidated.

   To ease comparison, this example will use the same function from the memoization step, this time caching the value
   for 15 minutes.

   ```php
   <?php

   function tec_fetch_user_visible_events(int $count, int $from, string $format): string {
       $cache = tribe_cache();
       $cache_key = __FUNCTION__ . $count . $from . $format . get_current_user_id();
    
       $cached = $cache->get( $cache_key );

       if( $cached ) {
           return $cached;
       }
        
       $events = tribe_events()->offset( $from )->per_page( $count )->get_ids() );

       $cache->set( $cache_key, $events, 900 );

       return $events;
   }
   ```

   You're done.

6. Cache the value until a post of a type controlled by The Events Calendar suite is created, updated or deleted

   The `Tribe__Cache__Listener::TRIGGER_SAVE_POST` will tell the cache to consider the cached value valid until a post
   of a type controlled by The Events Calendar suite (Events, Venues, Organizers, Tickets) is created, updated or
   deleted.

   Using the trigger in conjunction with an expiration time means: "until either a post is updated or this amount of
   time has passed"

   Caching will store the value in a single group that will be valid across all requests until invalidated.

   To ease comparison, this example will use the same function from the memoization step.

   ```php
   <?php

   function tec_fetch_user_visible_events(int $count, int $from, string $format): string {
       $cache = tribe_cache();
       $cache_key = __FUNCTION__ . $count . $from . $format . get_current_user_id();
    
       $cached = $cache->get( $cache_key );

       if( $cached ) {
           return $cached;
       }
        
       $events = tribe_events()->offset( $from )->per_page( $count )->get_ids() );

       $cache->set( $cache_key, $events, 900, Tribe__Cache__Listener::TRIGGER_SAVE_POST );

       return $events;
   }
   ```

   You're done.

7. Cache the value until until the main The Events Calendar option is updated

   The `Tribe__Cache__Listener::TRIGGER_UPDATED_OPTION` will tell the cache to consider the cached value valid until the
   main The Events Calendar option (`tribe_events_calendar_options`) is updated.

   Using the trigger in conjunction with an expiration time means: "until either the option is updated or this amount of
   time has passed"

   Caching will store the value in a single group that will be valid across all requests until invalidated.

   To ease comparison, this example will use the same function from the memoization step.

   ```php
   <?php

   function tec_fetch_user_visible_events(int $count, int $from, string $format): string {
       $cache = tribe_cache();
       $cache_key = __FUNCTION__ . $count . $from . $format . get_current_user_id();
    
       $cached = $cache->get( $cache_key );

       if( $cached ) {
           return $cached;
       }
        
       $events = tribe_events()->offset( $from )->per_page( $count )->get_ids() );

       $cache->set( $cache_key, $events, 900, Tribe__Cache__Listener::TRIGGER_UPDATED_OPTION );

       return $events;
   }
   ```

   You're done.

8. Cache the value until until rewrite rules are regenerated

   The `Tribe__Cache__Listener::TRIGGER_GENERATE_REWRITE_RULES` will tell the cache to consider the cached value valid
   until the main The Events Calendar option (`tribe_events_calendar_options`) is updated.

   Using the trigger in conjunction with an expiration time means: "until either rewrite rules are regenerated or this
   amount of time has passed"

   Caching will store the value in a single group that will be valid across all requests until invalidated.

   To ease comparison, this example will use the same function from the memoization step.

   ```php
   <?php

   function tec_fetch_user_visible_events(int $count, int $from, string $format): string {
       $cache = tribe_cache();
       $cache_key = __FUNCTION__ . $count . $from . $format . get_current_user_id();
    
       $cached = $cache->get( $cache_key );

       if( $cached ) {
           return $cached;
       }
        
       $events = tribe_events()->offset( $from )->per_page( $count )->get_ids() );

       $cache->set( $cache_key, $events, 900, Tribe__Cache__Listener::TRIGGER_GENERATE_REWRITE_RULES );

       return $events;
   }
   ```

   You're done.

[1]: ../whether-to-cache-or-not/index.md
