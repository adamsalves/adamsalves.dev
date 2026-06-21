# adamsalves.dev

Personal portfolio of **Adams Alves**, Front-End Engineer based in São Paulo, Brazil.
Built with [Hugo](https://gohugo.io/) and the custom **terminal-mono** theme. The site
is bilingual: Portuguese (default, served at `/`) and English (served at `/en/`).

🌐 Live: <https://adamsalves.dev>

## Tech stack

- **[Hugo](https://gohugo.io/)** (extended), `>= 0.158.0` — static site generator
- **[terminal-mono](https://github.com/adamsalves/terminal-mono)** — custom theme, included as a git submodule
- **Vercel** — hosting and CI (config in [`vercel.json`](./vercel.json))

## Getting started

The theme lives in a git submodule, so clone with `--recurse-submodules`:

```shell
git clone --recurse-submodules https://github.com/adamsalves/adamsalves.dev.git
cd adamsalves.dev
```

Already cloned without submodules? Pull them in:

```shell
git submodule update --init --recursive
```

Run the local dev server (requires the **extended** edition of Hugo):

```shell
hugo server
```

Then open <http://localhost:1313>.

Build the production site into `public/`:

```shell
hugo --gc --minify
```

## Project structure

```
.
├── content/        # markdown content (blog posts, per-language)
├── static/         # static assets served as-is (favicons, etc.)
├── archetypes/     # front-matter templates for `hugo new`
├── themes/
│   └── terminal-mono/   # theme (git submodule)
├── hugo.toml       # site config + all page content/params (en & pt)
└── vercel.json     # Vercel build configuration
```

Most of the page content (hero, about, experience, projects, contact) lives in the
`[languages.en.params.*]` and `[languages.pt.params.*]` tables in
[`hugo.toml`](./hugo.toml) — edit there to update the site copy.

## Deployment

The site is deployed on **Vercel** and rebuilds automatically on every push to the
default branch. Build settings are committed to the repo in [`vercel.json`](./vercel.json):

- `framework`: `hugo`
- `buildCommand`: `hugo --gc --minify`
- `outputDirectory`: `public`
- `HUGO_VERSION` is pinned via `build.env` so Vercel builds with the same Hugo
  version as local development. Bump it there when upgrading Hugo.

The theme submodule (`terminal-mono`) is public, so Vercel fetches it automatically
during the clone step — no extra configuration required.
