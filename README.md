# docs

[![docs](https://github.com/go-embedded-r/docs/actions/workflows/docs.yml/badge.svg)](https://github.com/go-embedded-r/docs/actions/workflows/docs.yml)
[![License](https://img.shields.io/badge/license-BSD--3--Clause-1F4E9C?style=flat-square)](LICENSE)

**The go-embedded-r documentation site**, published at
<https://go-embedded-r.github.io/docs/>.

A [Hugo](https://gohugo.io/) site with the [tannevaled/hextra](https://github.com/tannevaled/hextra)
fork of the [Hextra](https://github.com/imfing/hextra) theme (pinned to v0.16.0),
built the same way as the [go-embedded-julia documentation](https://github.com/go-embedded-julia/docs).
The organization is **in development**: milestones M0 (the oracle harness) and
M1 (the parser and deparse) are done, and M2 (the
evaluator) is done, in go-embedded-r/r v0.14.1; M3 (base R) is in progress and M4 to M8 are planned. The site
gives the plan, the parser and deparse with their parity tables against
R 4.6.2, the evaluator's design and slices, the Eval API, the
oracle harness, and the licence rule. Every statement comes from the README and
docs/m2-design.md of go-embedded-r/r on main.

## Versions

`params.libraries` in `hugo.yaml` names the release of each package the pages
describe, or `main` while it has no tag. A version of the documentation is
**the newest minor version among the tagged ones**, or `main` while none is
tagged:

| Source | Published under |
| --- | --- |
| branch `main`, with `params.libraries` in `hugo.yaml` | `https://go-embedded-r.github.io/docs/<MAJOR.MINOR>/`, or `/docs/main/` |
| the newest version (else `main`) | also `https://go-embedded-r.github.io/docs/latest/` |

`https://go-embedded-r.github.io/docs/` redirects to `latest/`. Each push to
`main` replaces the directory of the version it describes, and nothing else.
The interpreter is tagged v0.14.1, so the pages are published under `0.14/`,
which `latest/` follows; the older versions stay as published.

## Layout

| Path | What |
| --- | --- |
| `content/en/` | the pages; `_index.md` is the home, then the plan, the parser, deparse, the evaluator, the Eval API, regular expressions, the tre module, the M3 plan and base coverage, the clean-room process, files and connections, the oracle harness and the licences, `weight` orders the sidebar |
| `i18n/en.yaml` | the site's strings, beside the theme's |
| `hugo.yaml` | `baseURL`, the packages described, the navbar (version, theme, search, GitHub) |
| `layouts/_partials/custom/version-select.html` | the version selector (reads `versions.json`) |
| `layouts/_partials/custom/content-begin.html` | the outdated-version banner (reads `versions.json`) |
| `layouts/_partials/favicons.html` | the mark as favicon |
| `static/images/brand/svg/go-embedded-r.svg` | the mark (no brand repository yet) |
| `assets/css/custom.css` | the brand colour as Hextra's primary colour, the table headers |
| `themes/hextra` | submodule: [tannevaled/hextra](https://github.com/tannevaled/hextra), pinned to v0.16.0 |
| `scripts/publish-version.sh` | puts one build into a `gh-pages` checkout and rewrites `versions.json`, `latest` and the root redirect |

## Working locally

```sh
git clone --recurse-submodules https://github.com/go-embedded-r/docs.git
cd docs
mkdir -p data && sh themes/hextra/scripts/page-history.sh > data/pagehistory.json
hugo server --baseURL http://localhost:1313/docs/main/
```

then open <http://localhost:1313/docs/main/>. Hugo 0.146 or later; no Python,
no Node.

## Publication

`.github/workflows/docs.yml`:

| Job | When | What |
| --- | --- | --- |
| `build` | pull requests, `main` | builds with `--baseURL .../docs/<version>/`, checks internal links and anchors (lychee, offline), uploads the site |
| `links` | pull requests, `main` | fetches every external URL the pages name |
| `deploy` | `main` | `scripts/publish-version.sh` into `gh-pages` (started empty on the first deploy), then a commit and a push |

GitHub Pages serves the `gh-pages` branch.

## Licence

BSD-3-Clause.
