## Creating a throw-away plugin

A throw-away plugin is a plugin that is created for a specific purpose, and that is not meant to be used in a production
environment.

The plugin is barely functional, is not refined and will likely contain bugs and hard-coded values.
It should be cheap to create, and cheap to throw away.

Throw-away plugins are useful to test a specific feature, or to test a hypothesis, without having to modify an existing
plugin or theme.

### Steps

1. If your development site does not contain a `mu-plugins` directory, create it in the `wp-content` directory.

2. Create a new file in the `mu-plugins` directory, and give it a unique name, e.g. `throw-away-plugin.php`.

3. Add the following code to the file:

    ```php
    <?php
    /*
    Plugin Name: Throw-away Plugin
    */
   
    // Your code here
    ```

4. Run the request (e.g. visit the page) that you want to test and collect your results.

5. Once you have collected your results, you can delete the throw-away plugin.

### Notes

The `mu-plugins` directory is a special directory in WordPress that contains "Must Use" plugins. These plugins are
automatically activated and cannot be deactivated from the WordPress admin interface.  
The reason to put your throw-away plugin in the `mu-plugins` directory is that it will be automatically activated *
*before** any other plugin and will be available for all requests.

Here are some ideas for what you can do with a throw-away plugin:

* print to the `error_log` to debug a specific request
* add a filter by means of `add_filter` to test a behavior that would be hard to test otherwise
* add a new REST endpoint to test a specific feature
