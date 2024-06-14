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

1.1 If you want to add documentation for several versions of the repo (for example, main, 0.0.1, 0.0.2, 0.0.3), don't forget to checkout each tag specifically. For example, to checkout and document the latest tag, use the following logic:

```
- name: Checkout <new-repo> (latest release)
  uses: actions/checkout@v4
  with:
    repository:  <Owner>/<new-repo>
    path: <new-repo>-latest
    fetch-depth: 0

- name: Determine latest release commit for  <new-repo>
  run: |
    cd  <new-repo>-latest
    git fetch --tags
    LAST_TAG=$(git tag --sort=version:refname | tail -n 1 | head -1)
    echo "Checking latest out tag: $LAST_TAG"
    echo "$LAST_TAG" > LAST_TAG.txt
    git checkout $LAST_TAG

- name: Modify toc.yml with tags for <new-repo>
  run: |
    cd ${{ github.workspace }}
    TAG=$(cat <new-repo>-latest/LAST_TAG.txt)
    echo "Checking latest out tag: $TAG"
    sed -i "s|- name: framework latest release|- name: $TAG|g" toc.yml
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
      - name: <Second-level category name> (main)
        href: ./<src path from docfx.json>/README.html
      - name:  framework latest release
        href: ./<new-repo>-latest/README.html
```
