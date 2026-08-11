# go-gfx docs

Versioned documentation for [go-gfx](https://github.com/go-gfx/gfx), the pure-Go
2D graphics foundation. Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)
and versioned with [mike](https://github.com/jimporter/mike); published to the
`gh-pages` branch and served at **<https://go-gfx.github.io/docs/>**.

## Local preview

```sh
pip install -r requirements.txt
mkdocs serve
```

## Publishing

Pushing to `main` runs `.github/workflows/docs.yml`, which deploys the versioned
site with `mike deploy --push --update-aliases 0.1 latest` and sets `latest` as
the default. `/docs/` redirects to the default version.

## License

BSD-3-Clause — see [LICENSE](LICENSE). Copyright the go-gfx/docs authors.
