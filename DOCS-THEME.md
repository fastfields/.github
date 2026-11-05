# The shared documentation theme

The fastfields documentation is **one site assembled from four repositories**.
Each repository builds its own section, in its own repo, with its own toolchain
— there is no monolithic build, and **no docs build may fetch another
repository** (the API reference needs each package importable, so the builds
cannot be centralised).

```
/          fastfields.github.io   the landing page, the shared theme, the root
/python/   fastfields             the Python packages
/dlpack/   fastfields-dlpack      the raw DLPack binding
/lib/      fastfields-lib         the C++/CUDA library
/whl/      whl                    the PEP 503 wheel index
```

`/whl/` is special and deliberately left alone: the repo is named `whl`, so its
own project Pages site already serves at `fastfields.github.io/whl/`. It is a
package index consumed by `pip --extra-index-url`, not documentation, and it is
not part of the shared branch or this theme.

The other three publish into the **`gh-pages` branch of
`fastfields.github.io`**, which is what Pages serves.

## The invariant

> **A publisher writes only its own subtree.**

Several repositories deploying into one Pages site will delete each other's
directories if any of them does a full checkout-and-replace. So each section
syncs its own directory and nothing else; `tools/publish-docs.sh` (identical in
each publisher repo) is what enforces it, and the root publisher additionally
excludes every subtree it does not own.

Two further things worth knowing before changing any of it:

- **`concurrency:` groups are per-repository.** They cannot serialize
  publishers that live in different repos, so they are not the cross-repo
  guard — the retry-on-non-fast-forward loop in the publish script is. Keep the
  per-repo group anyway; it stops a repo racing itself.
- **`actions/deploy-pages` can only deploy to the repo it runs in**, and
  replaces the whole site. That is why the sections do not use it.

## Theming

**The stylesheet is a true single source.** `fastfields.github.io` owns
`docs/stylesheets/extra.css` and serves it at
`https://fastfields.github.io/stylesheets/extra.css`. Every other section
references that absolute URL:

```toml
extra_css = ["https://fastfields.github.io/stylesheets/extra.css"]
```

zensical passes an absolute URL through verbatim, so the browser fetches it at
page load. Nothing is fetched at build time, and the theme has one place to
change.

Its link icons match on the URL **path suffix** (`[href$="/lib/"]`), not on a
full absolute URL. A cross-section link is absolute while a link inside a
section is relative, and a path suffix matches both.

**The theme block is duplicated, because zensical has no include mechanism.**
Keep these identical in all four `zensical.toml` files:

```toml
[project.theme]
language = "en"
features = ["content.code.copy", "content.code.select", "navigation.indexes"]

[[project.theme.palette]]
scheme = "default"
toggle.icon = "lucide/sun"
toggle.name = "Switch to dark mode"

[[project.theme.palette]]
scheme = "slate"
toggle.icon = "lucide/moon"
toggle.name = "Switch to light mode"

[project.theme.icon]
logo = "fontawesome/solid/spiral"
repo = "fontawesome/brands/github"
```

## Nav

Every section's `nav` carries absolute links to the other three plus the wheel
index, so the four navigate as one site. Its own pages stay relative. For
example, from `/lib/`:

```toml
nav = [
  {"fastfields home" = "https://fastfields.github.io/"},
  {"Home" = "index.md"},
  {"Architecture" = "architecture.md"},
  {"API families" = "api.md"},
  {"Python packages" = "https://fastfields.github.io/python/"},
  {"Raw DLPack binding" = "https://fastfields.github.io/dlpack/"},
  {"Wheel index" = "https://fastfields.github.io/whl/"},
]
```

`site_url` must match where the section is actually served
(`https://fastfields.github.io/lib/`, not `/fastfields-lib/`) or the canonical
links and sitemap point at pages that do not exist.

## Adding a section

1. Give it a `zensical.toml` with the theme block above, the shared
   `extra_css`, the cross-section nav, and a `site_url` matching its path.
2. Copy `tools/publish-docs.sh` from any existing publisher, unchanged.
3. Add a `docs.yaml` that builds and then runs
   `./tools/publish-docs.sh --subdir <name> --site site`.
4. Give it `DOCS_DEPLOY_TOKEN` — `contents: write` on `fastfields.github.io`
   and nothing else. The default `GITHUB_TOKEN` is scoped to the calling repo
   and cannot push there.
5. Add the subtree's name to `OTHERS` in the **root** publisher's copy of the
   script, so the root deploy stops deleting it.

Step 5 is the easy one to forget, and the symptom is the new section vanishing
the next time the landing page is edited.
