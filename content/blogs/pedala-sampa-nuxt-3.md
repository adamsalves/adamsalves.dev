+++
title = "Migrating Pedala Sampa to Nuxt 3"
date = 2026-03-12
draft = false
description = "How I rewrote a Vue app to Nuxt 3 without losing SEO — routes, composables and a wayfinding redesign along the way."
tags = ["nuxt", "vue", "migration"]
toc = true
+++

In this post I share how I approached migrating Pedala Sampa to Nuxt 3: the decisions, the stumbles and what I'd do differently next time. No fluff — straight to what matters.

## Context

The starting point was simple on the surface, but full of details under the hood. I wanted a predictable base, easy to test, that wouldn't force me to relearn everything for each new feature.

The core idea was to isolate logic in `composables/` and keep components as dumb as possible — presentation only.

## Hands on

In practice, the heart of it lived in a single composable. Notice how state and actions come out together, ready for any component to consume:

```ts
// composables/useGroups.ts
export function useGroups() {
  const groups = ref<Group[]>([])

  async function load() {
    groups.value = await request(endpoint, GroupsQuery)
  }

  return { groups, load }
}
```

> Tip: start with the smallest step that delivers value. The rest comes later — and almost always simpler than you imagined.

The takeaways I kept from this snippet:

- State close to the logic, far from the UI.
- Typing early avoids 80% of the annoying bugs.
- A small test today is worth an afternoon of debugging tomorrow.

## Conclusion

In the end, what unblocks a project is rarely the newest technology — it's the decision to keep things simple and testable. If this post saved you a few minutes, it was worth it. 🚀
