# Arcane Docs

A repo that automatically creates documentation for Arcane, a Kubernetes-native data streaming service based on Akka.NET. The core of this project is a Github action named `release-documentation` that uses [DocFX](https://github.com/dotnet/docfx) which is designed to automate the process of generating and deploying documentation to GitHub Pages.


## Permissions

The action requires the following permissions:

- `contents: read`: To read the contents of the repository.
- `pages: write`: To write to the GitHub Pages site.
- `id-token: write`: To write ID tokens.

## Environment

The action runs on the latest version of Ubuntu.

## Contributing

Feel free to open a new [Issue](https://github.com/SneaksAndData/arcane-docs/issues) in case of any suggestions.
