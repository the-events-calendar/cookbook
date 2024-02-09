## Adding a new recipe

This recipe will guide you through the process of adding a new recipe to the cookbook.  

Before you start, make sure no existing recipe covers the same topic. If you are unsure, ask on Slack.

### Steps

1. Check out [the cookbook repository][1] using the command line:

   ```bash
   git clone git@github.com:the-events-calendar/cookbook.git
   ```
   
2. Create a new branch for your recipe:

   ```bash
    git checkout -b my-new-recipe
    ```
   
3. Create a new directory for your recipe and add an `index.md` file:

   ```bash
    mkdir recipes/my-new-recipe
    touch recipes/my-new-recipe/index.md
    ```
4. Add your recipe content to the `index.md` file. Use the following template:
    
    ```markdown
    ## Recipe Title

    A brief description of the recipe.

    ### Steps

    1. Step 1
    2. Step 2
    3. Step 3
  
    ### Notes
    
    Any additional notes or considerations.
    This is a good place to add links to related resources.
    ```
   
5. When you're satisfied with your recipe, commit your changes, and push your branch to the repository:

   ```bash
    git add .
    git commit -m "Add my new recipe"
    git push origin my-new-recipe
    ```
   
6. Open a pull request on GitHub. Make sure to add a clear title and description for your pull request.

7. Once your pull request is approved, merge it into the `main` branch.
   
### Notes
Keep the notes short and to the point. If you need to add more information, consider creating a new recipe and link to it from the recipe you are working on.  
Recipes are written to guide engineers through a process. They should be clear and concise.  

The cookbook repository structure is based on the WordPress [`developer-plugins-handbook`][2] repository to be able to render the recipes in a site like the [WordPress Developer Handbook][3] one.  

Use Markdown to write your recipe. If you are not familiar with Markdown, you can find a guide [here][3].

When it comes to links, do **not** use inline links. Instead, use reference links. For example:

```markdown
Visit my awesome [site][1]

[1]: https://example.com
```

Prefer providing code in fenced code blocks over images of code.  

If you think adding an image is necessary, place it in the `images` directory of your recipe and reference it in your recipe using a relative path.  

```markdown
![Alt text](images/my-image.png)
```

[1]: https://github.com/the-events-calendar/cookbook
[2]: https://github.com/WordPress/developer-plugins-handbook
[3]: https://developer.wordpress.org/plugins/shortcodes/
[4]: https://www.markdownguide.org/basic-syntax/
