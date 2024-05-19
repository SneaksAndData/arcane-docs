# Arcane Docs

A repo that automatically creates documentation for Arcane, a Kubernetes-native data streaming service based on Akka.NET. The core of this project is a Github action named `release-documentation` that uses [DocFX](https://github.com/dotnet/docfx) which is designed to automate the process of generating and deploying documentation to GitHub Pages.

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

## Permissions

The action requires the following permissions:

- `contents: read`: To read the contents of the repository.
- `pages: write`: To write to the GitHub Pages site.
- `id-token: write`: To write ID tokens.

## Environment

The action runs on the latest version of Ubuntu.

## Contributing

Feel free to open a new [Issue](https://github.com/SneaksAndData/arcane-docs/issues) in case of any suggestions.
