# go-gfx docs

Versioned documentation for [go-gfx](https://github.com/go-gfx/gfx), the pure-Go
2D graphics foundation. Built with [Hugo](https://gohugo.io/) and this fleet's [Hextra fork](https://github.com/tannevaled/hextra) — upstream Hextra plus the opt-in features the fleet's sites use; published to the
`gh-pages` branch and served at **<https://go-gfx.github.io/docs/>**.

## Local preview

```bash
git submodule update --init --recursive   # themes/hextra, pinned at a release tag
hugo server                               # http://localhost:1313
```

## Publishing

Pushing to `main` runs `.github/workflows/docs.yml`, which deploys the versioned
site with `mike deploy --push --update-aliases 0.1 latest` and sets `latest` as
the default. `/docs/` redirects to the default version.

## License

BSD-3-Clause — see [LICENSE](LICENSE). Copyright the go-gfx/docs authors.
