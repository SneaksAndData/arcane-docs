## TBD:

1. Create documentation for multiple repos
2. Create a dropdown menu so a user can navigate between Arcane's components
3. Versioning navigation for users (main vs latest tag)
4. An action should be run on deployment, not on PR
5. An index page with human-friendly description and use cases
6. Instructions on how to add a repository to the generation process
7. Update hostname

- api: this is where DocFx creates documentation based on source code, e.g XML comments or csproj files
- apidoc: this one contains Markdown files where we can override the XML comment defaults
- articles: this is non-code documentation, for example, how-to guides or engineering guidelines
- images: if we want to add images to our Markdown files, we can add them here
- src: this is where we can place our CSPROJ project files used to generate source code

Let’s now highlight each important file:

- docfx.json: this is the DocFx configuration file. We can use it to make general configuration articles, which we’ll do later in the article
- index.md: the main page, written in markdown
- toc.yml: the table of contents for our documentation site

https://dotnet.github.io/docfx/docs/table-of-contents.html

https://blog.markvincze.com/build-and-publish-documentation-and-api-reference-with-docfx-for-net-core-projects/
