## Updating a recipe

This recipe will guide you through the process of updating an existing recipe in the cookbook.

### Steps

1. Check out [the cookbook repository][1] using the command line:

   ```bash
   git clone git@github.com:the-events-calendar/cookbook.git
   ```

2. Create a new branch for your recipe update:

   ```bash
    git checkout -b update-my-recipe
    ```
   
3. Navigate to the directory containing the recipe you want to update and edit it following the recipe guidelines outlined in the [Adding a recipe][2] recipe.

4. When you're satisfied with your updates, commit your changes, and push your branch to the repository:

   ```bash
    git add .
    git commit -m "Updating my recipe"
    git push origin update-my-recipe
    ```

5. Open a pull request on GitHub. Make sure to add a clear title and description for your pull request.

6. Once your pull request is approved, merge it into the `main` branch.

[1]: https://github.com/the-events-calendar/cookbook
[2]: ../adding-a-new-recipe/index.md
