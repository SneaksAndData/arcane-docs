## TBD:

[x] Create documentation for multiple repos
[x] Create a dropdown menu so a user can navigate between Arcane's components
[] Versioning navigation for users (main vs latest tag)
[] An index page with human-friendly description and use cases (?)
[] Instructions on how to add a repository to the generation process
[] Update hostname (?)
[] An action should be run on deployment, not on PR

## Useful info:

- api: this is where DocFx creates documentation based on source code, e.g XML comments or csproj files
- apidoc: this one contains Markdown files where we can override the XML comment defaults
- articles: this is non-code documentation, for example, how-to guides or engineering guidelines
- images: if we want to add images to our Markdown files, we can add them here
- src: this is where we can place our CSPROJ project files used to generate source code

### Important files:

- docfx.json: this is the DocFx configuration file. We can use it to make general configuration articles, which we’ll do later in the article
- index.md: the main page, written in markdown
- toc.yml: the table of contents for our documentation site

[Official documentation](https://dotnet.github.io/docfx/docs/table-of-contents.html)
