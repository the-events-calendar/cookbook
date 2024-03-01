## TODO Recipes

This file is a collection of ideas for new recipes while the cookbook is its initial stages.  
Eventually, this file will be removed and replaced.

In the meantime, if you have an idea for a new recipe, and do not have the time to [Add a new recipe][1], add it here.

### TODO

* Fetch data from TEC + CT1 custom tables using models
* Refactor use of `Tribe__Date_Utils` to use the `StellarWP\Dates\Dates` class
* Refactor uses of a mutable date objects to use immutable date objects
* Fetch a value from cache and protect against cache poisoning
* Fetch an unbound number of posts from the database
* Hook on a filter or action from WordPress core or a third-party plugin
    - Loose typing
* Hook on a filter or action from one of TEC plugins
    - Strict typing
    - Loose typing in _some_ instances
* Remove a class method
* Update a class method signature
* Remove a class
* Add a feature
* Refactor a Service Provider to make it a Controller
* Refactor code that hooks on filters to make it a Controller
* Test the integration of a TEC or ET feature with one or more premium plugins
* Test the integration of a plugin feature with a third-party plugin
* Test the integration of a plugin feature with a version of WordPress that is not the minimal supported one
* Add a new integration test suite
* Add a new test suite
* Create Single or Recurring Events in tests
* Test code that produces HTML, JSON or other outputs that contain post IDs
    - AUTOINCREMENT trick
    - no mocks or post redirection
* Refactor test code that uses post redirection to use real posts
* Debug tests behaving differently when running alone vs when running with other tests
    - Run them in larger and larger scopes
* Debug failures in CI that you cannot reproduce locally
    - Check the hash/version of everything
* Decide how to test a piece of code
* Setup or debug XDebug connection to PHPStorm to debug `slic` tests
* Setup or debug XDebug connection to VS Code to debug `slic` tests
* Setup or debug XDebug connection to PHPStorm from a Lando server
* Setup or debug XDebug connection to VS Code from a Lando server
* Write an integration with a third-party plugin in TEC or ET
* Write an integration with a third-party plugin in a premium plugin
* Check if a function or method is covered by tests
* Fix a bug in existing code that is not covered by tests
* Test code that sets constants, or requires constants to be se
* Test code that relies on database transactions
* Setup the debug compiled JS code in PHPStorm
* Setup the debug compiled JS code in VS Code
* Find out if a class is used
* Find out if a class method is used
* Find out if a function is used
* Architect a feature and present an architecture document
    - Find the 99% and build back from it
    - Test and research the 99% with low-effort, ephemeral artifacts (curl, wp shell, manual queries, throw-away plugins)
    - Back-compatibility


[1]: recipes/adding-a-new-recipe/index.md
