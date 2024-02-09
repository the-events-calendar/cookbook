## Refactor an unbounded TEC ORM query

The TEC ORMs are a set of functions that allow querying the database for the post types defined by The Events Calendar,
Event Tickets, and other plugins in the suite.

While trying to do "the right thing", the ORMs can still be misused to generate unbounded queries that usually implement
a use case that sounds like "get all the things that match a certain criteria".

In those instances, the use of the ORMs should be refactored to either include a limit, or to use a method returning a
Generator to fetch the results in chunks.

This recipe assumes you've found the unbounded query source in the codebase, and you want to refactor it to a bounded
query.

### Steps

1. Can you paginate the results? Do you **really** need all the results at once? If you can paginate the results, use
   the `per_page( int $per_page )` method of the ORM to fetch the results in chunks:

    ```diff
    - $events = tribe_events()->all();
    + $events = tribe_events()->per_page( 10 )->all();
    ```
   If this change is not possible, go to the next step.

2. Are you fetching all the elements to count them? If so, use the `count()` method of the ORM to fetch the count of the
   results, instead of fetching all the results and counting them in PHP:

    ```diff
    - $ids = tribe_events()->where( 'status', 'private' )->get_ids();
    - $count = count( $ids );
    + $count = tribe_events()->where( 'status', 'private' )->count();
    ```
   If this is not applicable, go to the next step.

3. If you need all the results at once, set the flag, in the `all()` and `get_ids()` methods, to fetch the results in
   chunks, using a [Generator][1]:

    ```diff
    - $events = tribe_events()->all();
    + $events = tribe_events()->all( true );
   
    foreach( $events as $event ) {
        // Do something with the event.
    }
    ```
   Or with `get_ids()`:
    ```diff
    - $ids = tribe_events()->get_ids();
    + $ids = tribe_events()->get_ids( true );
   
    foreach( $ids as $ids ) {
        // Do something with the event ID.
    }
    ```
   Under the hood, the methods will fetch the values in chunks, and return a [Generator][1] that will allow you to
   iterate over the results.

### Notes

In the example code you see use of the `where()` method to filter the results. The `where()` method is an alias of
the `by()` one, they are functionally equivalent.

In your refactoring, do not "unroll" the Generator, as that would defeat the purpose of fetching the results in
chunks. Instead, use the Generator to iterate over the results.

   ```diff
    - $all_events = iterator_to_array( tribe_events()->all( true, 10 ) );
    + $all_events = tribe_events()->all( true, 10 );
   
    foreach( $all_events as $event ) {
        // Do something with the event.
    }
   ```

This approach will only fetch 10 events at a time, and will hold, at most, the data of 10 events in RAM.

[1]: https://www.php.net/manual/en/language.generators.overview.php
