+++
title = "Hello, world: the portfolio is live — and a Hugo theme came out of it"
date = 2026-08-11
draft = false
description = "I shipped adamsalves.dev, tidied up my GitHub profile, and the theme that came out of the project ended up as terminal-mono, released under the MIT license."
tags = ["hugo", "portfolio", "open-source"]
toc = false
+++

First post on the blog, so let me start at the start: [adamsalves.dev](https://adamsalves.dev/) is live. I set out to build a decent place to keep my projects and ended up with three things instead.

## The site

It's a single-page portfolio — a terminal-shaped hero, projects in cards that look like repositories, experience, contact and this blog. Built with Hugo, published on Vercel, static from end to end, and in two languages: Portuguese at `/`, English at `/en/`.

The idea was to get away from the generic template without going overboard: dark theme, monospace type, a lime accent and no third-party JS. All the section content lives in `params`, in `hugo.toml` — so updating the site means touching config, not templates. That was deliberate: templates are the kind of thing you edit once and then forget how they work.

## The GitHub profile

I took the chance to tidy up my [GitHub](https://github.com/adamsalves) too. The README now has a short summary of what I do and the stack I work with day to day (React, Vue 3, TypeScript, Next.js, Nuxt, React Native, Node.js — plus Go, which is next on my list), and I pinned the projects I actually enjoy showing people:

- **pedala-sampa**, a collaborative map for São Paulo cyclists, built with Vue 3 / Nuxt, Leaflet and GraphQL;
- **phantom-collector**, an 8-bit arcade game in Phaser 3 where the chiptune audio is generated on the fly by the Web Audio API;
- **planning-pvoker**, real-time planning poker over WebSockets, with consensus stats;
- **terminal-mono**, this site's theme — which is what the next section is about.

## terminal-mono

That one wasn't part of the plan. Somewhere along the way I noticed the theme didn't depend on anything of mine: all the content comes from `params`, and the colors and fonts come from CSS variables. For someone else to use it, all it took was editing a config file. At that point, keeping it locked inside this repo made little sense.

So I split it into its own repository, wrote the docs, put together a bilingual `exampleSite` to use as a reference, and released it under the MIT license: [terminal-mono](https://github.com/adamsalves/terminal-mono) — with [tagged releases](https://github.com/adamsalves/terminal-mono/releases), in case you'd rather pin a specific version.

What's already sorted: the hero with its typewriter effect, which degrades gracefully when there's no JS; the whole blog, with pagination, table of contents, tags, a reading-progress bar and sharing; i18n with English and Portuguese included; SEO (Open Graph, JSON-LD, RSS, `canonical`); accessibility (skip link, visible focus, `prefers-reduced-motion`); and Hugo's asset pipeline with minify, fingerprint and SRI in production.

Installing it is quick, through Hugo Modules:

```bash
hugo mod init github.com/your-name/your-site   # if you don't use modules yet
hugo mod get github.com/adamsalves/terminal-mono
```

```toml
# hugo.toml
[module]
  [[module.imports]]
    path = "github.com/adamsalves/terminal-mono"
```

It needs Hugo **extended** ≥ 0.158. If you'd rather use a git submodule, that works too.

## What's next

The blog starts here. I plan to write about the things that come up in day-to-day front-end work — design systems, testing, Vue and React — and about the behind-the-scenes of my projects, which is usually the more interesting part. And if terminal-mono turns out to be useful to you, open an issue, send a PR, or just tell me what broke.
