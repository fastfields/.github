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

## Credentials

`fastfields.github.io` publishes its own site root, so the workflow's default
`GITHUB_TOKEN` is enough there (`--auth token`).

The other three push into a repo that is not their own, where `GITHUB_TOKEN`
does not reach. They use **`DOCS_DEPLOY_KEY`**, an organization secret scoped
to `fastfields`, `fastfields-lib` and `fastfields-dlpack`: the private half of
an **SSH deploy key** whose public half is registered on `fastfields.github.io`
with write access. So:

- the remote is **`git@github.com:fastfields/fastfields.github.io.git`**, over
  SSH — an `https://x-access-token:...@github.com/...` remote will *not*
  authenticate with a deploy key;
- the key is written to a mode-600 file and used via `GIT_SSH_COMMAND`, which
  keeps a third-party action out of the deploy path;
- a deploy key authenticates **as the repository, not as a user**, so the
  commit identity is set explicitly to
  `fastfields-docs-bot <noreply@github.com>`, or the commit has no usable
  author.

The script checks the secret is non-empty **before doing any work** and exits 1
with one line naming `DOCS_DEPLOY_KEY` and the three repos it must be visible
to. A deploy key cannot expire, but it can be revoked or scoped wrong, and this
project has already lost weeks to a credential that stopped working silently.

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

**Every section declares the same nine top-level entries, these labels, this
order, and nothing else at the top level.** One entry per fastfields namespace,
plus `Home` and the wheel index:

```
Home
fastfields.auto
fastfields.numpy
fastfields.torch
fastfields.cupy
fastfields.helpers
fastfields.dlpack
libfastfields
Wheel index
```

That is the canonical menu. It is what makes the four sections read as one
site instead of four, and it is checked — see [Keeping the nav
honest](#keeping-the-nav-honest) below.

The namespace order is the order of the **Packages table on the landing page**.
Keep the two consistent; if they ever disagree, they are both wrong until
someone decides which.

### The rules

- **A section may expand any of the nine into a nested list of its own pages.**
  That is the point of a shared top level — it stays recognisable while each
  section fills in what it actually documents. From `/python/`:

  ```toml
  {"fastfields.numpy" = [
    {"Overview" = "numpy.md"},
    {"API reference" = "api/numpy.md"},
  ]},
  ```

  The nested labels are the section's own business. Only the top level is
  fixed.

- **No section may add, remove, reorder or relabel a top-level entry.** A page
  that is not a namespace — `/python/`'s `monorepo.md`, `/lib/`'s
  `architecture.md` and `api.md`, everything under `/lib/legacy/` — **nests
  under one of the nine**. It does not sit beside them. A section with no
  obvious parent for a page should ask which namespace owns it, because the
  answer is usually clear and the alternative is drift.

- **A target on the current site is a relative path; a target on another
  section is the absolute `https://fastfields.github.io/…` URL.** This is what
  lets one menu render identically everywhere while keeping in-site navigation
  native: a relative link gets the correct active-page highlight and no full
  page reload, and zensical passes an absolute URL through verbatim. So `Home`
  is `index.md` on the root and `https://fastfields.github.io/` everywhere
  else, and each section spells its own namespace entries relatively and the
  rest absolutely.

- **There is no separate "fastfields home" entry.** `Home` is entry one and
  does that job.

### `site_name` and `site_url`

- **`site_name` is `"fastfields"` in every config**, including
  `fastfields-lib`'s. Every section documents one product, and the header is
  not the place to say which part of it you are in — each page's own `#`
  heading does that. (`fastfields-lib` set `"fastfields-lib"` here for a while,
  which made the header read as a different product on a third of the site.)
- **`repo_url` / `repo_name` stay pointed at each section's own repository.**
  The header's GitHub link should go to the code being documented, so these are
  deliberately *not* unified.
- **`site_url` must match where the section is actually served**
  (`https://fastfields.github.io/lib/`, not `/fastfields-lib/`) or the
  canonical links and the sitemap point at pages that do not exist.

### Keeping the nav honest

`tools/check-docs-nav.py` parses a `zensical.toml`, extracts the top-level nav
labels in order, and fails if they are not exactly the nine above (it also
checks `site_name`). It is **identical in all four publisher repos and each one
runs it against its own config**, from the `nav` job in that repo's
`docs.yaml`, on every push *and every pull request*. A check living in only one
repository cannot see another repository's drift, which is precisely how four
different menus came to exist.

`--selftest` checks the checker against known-good and known-broken navs, so a
guard that has quietly stopped failing on anything cannot pass for a working
one. CI runs it before the real check.

Changing the canonical menu therefore means changing `CANONICAL_NAV` in four
copies of the script, this document, and four `zensical.toml` files — on
purpose. The menu is a cross-repository contract and it should cost something
to move.

## Adding a section

1. Give it a `zensical.toml` with the theme block above, the shared
   `extra_css`, `site_name = "fastfields"`, the canonical nav, and a `site_url`
   matching its path.
2. Copy `tools/publish-docs.sh` and `tools/check-docs-nav.py` from any existing
   publisher, both unchanged.
3. Add a `docs.yaml` with a `nav` job that runs
   `tools/check-docs-nav.py --selftest` and then
   `tools/check-docs-nav.py --check zensical.toml` (on pushes *and* pull
   requests), and a `build-deploy` job that `needs: nav`, builds, and runs
   `./tools/publish-docs.sh --subdir <name> --site site`.
4. Make sure the `DOCS_DEPLOY_KEY` organization secret is scoped to it, and
   invoke the script with `--auth ssh`. The default `GITHUB_TOKEN` cannot be
   used: it is scoped to the repo it runs in.
5. Add the subtree's name to `OTHERS` in the **root** publisher's copy of the
   script, so the root deploy stops deleting it.

Step 5 is the easy one to forget, and the symptom is the new section vanishing
the next time the landing page is edited.
