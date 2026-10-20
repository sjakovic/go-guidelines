# Go Engineering Guidelines

How I structure and write Go services, CLIs and desktop apps: layout, packages, interfaces, errors, HTTP API conventions, database access, testing, size limits and tooling.

The guideline is written to be copied into a project as `docs/GO_GUIDELINES.md` and adjusted. It is the document behind the blog post [How I Structure Go Projects (After Two Real Ones)](https://jakovic.com/articles/how-i-structure-go-projects-after-two-real-ones).

- [GO_GUIDELINES.md](GO_GUIDELINES.md) - the guideline
- Issues and pull requests are welcome if you disagree with a rule and can say why.

## Credits

The rules stand on the shoulders of the Go team's [Organizing a Go module](https://go.dev/doc/modules/layout), the [Google Go Style Guide](https://google.github.io/styleguide/go/), Ben Johnson's "Standard Package Layout", Dave Cheney's "SOLID Go Design" and "Don't just check errors, handle them gracefully", and Mat Ryer's "How I write HTTP services in Go after 13 years".
