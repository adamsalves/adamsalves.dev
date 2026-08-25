+++
title = "The font was the critical path: auditing terminal-mono"
date = 2026-08-25
draft = false
description = "I opened an audit against my own theme. The Google font cost 1.6 s on a cold visit, the secondary text token failed WCAG in all five palettes, and the site had nothing to say to an answer engine."
tags = ["hugo", "performance", "aeo", "open-source"]
toc = true
+++

In the [previous post](/en/blogs/hello-world/) I wrote about this site's theme becoming its own repository. It's at v0.6.0 now, and nearly everything that landed over the last few weeks came out of an audit I opened against myself: Lighthouse 12.8 against the published `exampleSite`, and the [aeo.js](https://aeojs.org/) checker against the same URL.

The starting numbers weren't bad. Performance between 91 and 93, accessibility 95, SEO 100. But three audits came back at zero on all three pages I measured — `render-blocking-resources`, `render-blocking-insight` and `network-dependency-tree-insight` — and AEO scored 55 out of 100, with Schema Presence at zero.

What held the score down wasn't the JavaScript. TBT was 0 ms and the DOM had 201 elements. It was the font.

## The font was the critical path, not the theme's CSS

`head.html` loaded JetBrains Mono with a `<link rel=stylesheet>` to `fonts.googleapis.com`. That builds a serial chain the browser only discovers after parsing the HTML:

```
localhost/                                  706 ms
 └─ fonts.googleapis.com/css2?family=…      694 ms   ← render-blocking
     └─ fonts.gstatic.com/….woff2         1,602 ms   ← only starts here
```

`terminal.min.css` downloaded in parallel and finished at 1,261 ms. So the font was the critical path and the theme's own stylesheet was not. The two `preconnect` hints already there cover the handshake and don't remove the hop: the browser still has to fetch and parse Google's CSS before it knows which `.woff2` to ask for.

Lighthouse put the saving from removing the block at 1,330 to 1,590 ms, depending on the page.

I self-hosted it. Two subsets, `latin` at 31.4 KB and `latin-ext` at 11.6 KB, both variable fonts covering `400 800` — one file per subset serves every weight the theme uses. Only `latin` gets preloaded, because a preload the page never asks for is a wasted round trip and a console warning.

One thing I didn't expect to find on the way: the old URL asked for `wght@400;500;600;700;800`, and Google answered with **30 `@font-face` blocks and 12.4 KB of CSS** to arrive at the same file `wght@400..800` returns in six.

Measured after: mobile **92 → 100**, FCP **2.5 s → 1.1 s**, Speed Index **4.4 s → 1.1 s**, and the longest critical chain **1,602 ms → 44 ms**. The three zeroed audits pass.

The binaries are the ones `fonts.gstatic.com` serves for v24, committed unmodified, with a `SHA256SUMS` file CI verifies. That isn't decorative care: the `immutable` cache header I recommend in the README is only safe if replacing the bytes means writing a different filename. Swapping them in place now fails the build instead of poisoning the cache of everyone who already visited.

## The typewriter rebuilt the same node 442 times

The hero types 442 characters. Every tick wrote `tailEl.innerHTML`, and `wrap()` for an unfinished segment always produces the same shape — one span with one color. That was 442 trips through the HTML parser to rebuild a node identical to the one it had just destroyed.

The tail is now one span holding one text node, both built once, with the loop reassigning `.data`. Over three Lighthouse runs on a production build, mobile: **HTML parsing 62 ms → 12 ms**, consistently and well outside the run-to-run spread.

The loop also moved from `setTimeout(…, 15)` to `requestAnimationFrame`, writing once per frame instead of once per character — and only in the frames where a character actually came due, since assigning the same string still marks the node dirty, and the opening pause is some twenty-five frames of exactly that. It stops entirely in a background tab, where the old timer kept typing to nobody.

And `.term__body` got `contain: layout style`. The `min-height` above it already guarantees the box doesn't change size while it fills; this is telling the browser so. Without it, every write invalidated the layout of the whole document — 172 invalidations across one run.

## Two things didn't survive being measured

This is the part I nearly left out, and the part I'd most have wanted to read somewhere else first.

The original report attributed **1.4 s of main thread** to the typewriter. That number came from an A/B against `--force-prefers-reduced-motion`, which doesn't run the animation at all. So it's the cost of the animation **existing**, not the cost of how it's written. The rewrite is worth the 50 ms of parsing above plus the work it stops doing in a hidden tab. It isn't worth 1.4 s, and I'd have written that it was if I hadn't measured again.

`forced-reflow-insight` doesn't clear either. It fires on about half of Lighthouse's runs both before and after, because what's forced is the document's first layout, which the hero's height reservation genuinely needs. I tried reusing one persistent probe instead of building one per call: it moved the audit not at all, and it left the ruler's `0123456789` inside the terminal's `textContent` for the life of the page, readable by anything that extracts text from the rendered DOM — which on this theme now includes answer engines. Reverted. Deferring the first measurement into `requestAnimationFrame` would clear the audit by giving up the reservation, which is the layout shift the reservation exists to prevent.

Both are recorded in the changelog rather than quietly dropped.

## `--dim` failed WCAG in all five palettes

Lighthouse flagged two occurrences, `.term__title` on the home page and `.toc__label` on a post. But the problem was the token, not the two selectors: `--dim` is read by thirteen rules in `terminal.css` — the footer, post metadata, the pager, timeline dates, the language switcher, `figcaption`, Chroma's line numbers. All of it text at 12 to 13.5 px, where the threshold is 4.5:1 rather than the 3:1 large text gets.

It measured between 3.44:1 and 4.07:1 depending on the palette, and the worst case was against `--surface-2` — the **lightest** ground, not `--bg`. None of the five cleared it against any ground.

I lifted each value the minimum needed to clear 4.5:1 against that worst case, preserving the hue: `#6b7263 → #798071` for lime, `#776a56 → #887b67` for amber, `#6d6390 → #8076a3` for cyberpunk, `#62737f → #6e7f8b` for ice, `#6a6a6a → #7d7d7d` for mono. Accessibility went to 100 on every page.

Two of those fixes weren't in the report and turned up because I checked the whole ramp instead of the flagged token: mono's `--dim-2` was failing the same way at 4.45:1, and lifting amber's `--dim` pushed it **above** the `--dim-2` it's named to sit under — a palette that passes AA while contradicting its own token names.

A `check_contrast.py` went into CI alongside it. It asserts three properties of all five palettes at once: every text token clears 4.5:1 against every declared ground, the ramp `--dim < --dim-2 < --muted-2 < --muted < --prose < --soft < --text` is ordered by luminance, and any value the parser can't read is a named failure rather than a token it walks past. That last one is the checker checking itself: a `--dim` written `rgb(106,106,106)` never entered the parser at all, so a stylesheet failing AA came back clean.

## The theme didn't say what it was

The other half of the audit was AEO — Answer Engine Optimization, the new acronym for "what happens when the thing reading your page is a model answering someone's question."

The diagnosis was short and ugly:

- `robots.txt` was Hugo's default stub, literally `User-agent: *` and nothing else, with no `Sitemap:` line;
- there was no `llms.txt` and no `llms-full.txt`, 404 on both;
- JSON-LD came out on the home page only, and only as `Person`. Posts never said they were posts. That's where the zero on Schema Presence came from.

The theme builds with `hugo` and nothing else, and aeo.js is a 6.5 MB npm package with no Hugo integration — adopting it as a build dependency would mean bringing Node in, and the "Human/AI" widget it injects contradicts the "no third-party JS" that sits in the README, in `theme.toml` and in the repository description. So I implemented the outputs natively in Hugo and use the checker only as a checker.

`robots.txt` now names the AI crawlers in two groups, because they aren't the same request. An answer engine fetches the page to answer a question now and hands the citation back to the reader: `OAI-SearchBot`, `Claude-User`, `PerplexityBot`, `Applebot` and six more. A dataset crawler collects the page into a corpus, with no citation and no referral: `GPTBot`, `ClaudeBot`, `Google-Extended`, `CCBot` and four more. `[params.aeo] allowAI` and `allowTraining` switch the two independently, both defaulting to `true` — which is what the bare `User-agent: *` already meant, so upgrading the theme doesn't change what your site publishes behind your back.

robots.txt groups don't inherit, and that matters: a path excluded only from `*` would stay open to exactly the crawlers you just named. So `disallow` is repeated into every allowed group.

JSON-LD now covers what the theme actually renders — `BlogPosting` on posts, `WebPage` on other single pages, `CollectionPage` on the blog index and the tag pages, `BreadcrumbList` on everything but the home. The nodes are linked rather than repeated: the publisher carries an `@id` and the post's `author` points at it. Breadcrumbs are built from `.Ancestors`, the real content tree, not the URL string — so a crumb can't point at something that isn't a page.

The 404 is the one page that emits nothing. It isn't in the content tree, so a breadcrumb there describes a hierarchy that doesn't contain it, and a `WebSite` node invites a crawler to treat an error as a document.

One thing only surfaced when I sat down to write the test: `truncate` escapes a plain string and leaves a `template.HTML` alone. `headline` is the one field the theme transforms rather than copies, and it came out as `Vue &amp;amp; Vitest` for a title as ordinary as `Vue & Vitest` — HTML entities inside a JSON string, where the consumer reads them literally, contradicting the `name` built from the same title in the same node. `name` had the opposite problem, carrying whatever markup the front matter wrote.

## llms.txt, and what the spec actually asks for

The three files are Hugo Output Formats. `/llms.txt` is the index an answer engine can read in one request instead of crawling: title, summary, and every post as a linked item with a line of context. `/llms-full.txt` is the content behind those links in one file, each post preceded by its canonical URL. And each post publishes an `index.md` next to its HTML.

All of it per language. This site publishes `/llms.txt` and `/en/llms.txt`, each listing its own posts.

Two decisions that looked like details and weren't:

`llms.txt` links to the markdown twins rather than the HTML, which is what the spec asks for — and the canonical URL is the first line inside each twin, so a citation that follows the link still knows where to point.

Posts in `llms-full.txt` are separated by a `--- post: <url> ---` line rather than a bare `---`. A thematic break is ordinary markdown that someone writes inside a post without thinking about this file at all, and it was indistinguishable from the line that separates two posts. So was a setext underline under a heading.

And all three honour `[params.aeo] disallow` and the same build condition `robots.txt` uses. A path excluded from crawlers whose full body sits in `llms-full.txt` is not excluded, and answer engines are the audience that key names.

## The checker gets things wrong too

Worth saying, because I lost time to it: part of the distance between 55 and 100 was the checker's bug, not the theme's.

aeo.js failed "Canonical URL set", "JSON-LD schema found" and "Description length" — all three correct on the site. Its regexes require quoted attributes, and `hugo --minify` emits `rel=canonical` unquoted, which is valid HTML5. It also scans `new URL(target).origin`, so a site published under a sub-path gets scored on files it can't even see.

The score prints to the deploy job's summary with those two limits printed next to it. As information, never as a gate. The check that counts is the repo's own `check_aeo.py`, which runs against the bytes about to go out.

## Where this is now

v0.6.0 is published, this site already runs it, and the issue that pulled all of this together is closed. The theme still has `hugo` as its only build requirement and makes no third-party request — now including the font.

If you use terminal-mono and something here broke on your end, [open an issue](https://github.com/adamsalves/terminal-mono/issues). The full list of changes, with the numbers and the parts that didn't survive being measured, is in the [CHANGELOG](https://github.com/adamsalves/terminal-mono/blob/main/CHANGELOG.md).
