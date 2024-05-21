## Adding a new repository

To add a new repository to the documentation generation process, follow the steps:

1. Add a new checkout step in the GitHub Action workflow:

```
- name: Checkout <new-repo>
 uses: actions/checkout@v4
 with:
   repository: <Owner>/<new-repo>
   path: <new-repo>
```

2. Update the `docfx.json` configuration file by adding a new item to `metadata`:

```
{
  "src": [
    {
      "src": "<path_to_source_directory>",
      "files": [
         "**/*.csproj"
      ]
    }
  ],
  "dest": "<destination_directory>"
}
```

3. Add a new menu item in the `toc.yml ` file:

```
items:
  - name: Home
    href: ./index.html
  - name: <New tab (primary category)>
    items:
      - name: <Second-level category name>
        href: ./<src path from docfx.json>/README.html
```
