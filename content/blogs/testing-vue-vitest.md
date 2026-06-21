+++
title = "Testing Vue in practice: Vitest + Testing Library"
date = 2026-01-21
draft = false
description = "A TDD flow I actually keep: what to test, what to skip, and how to keep the suite fast."
tags = ["testing", "vitest", "tdd"]
+++

Testing Vue became enjoyable with Vitest. But a good tool doesn't replace judgment: what's actually worth testing?

## What to test

I test **behavior**, not implementation. If the user clicks and something shows up, that's what the test checks.

```ts
import { render, screen } from "@testing-library/vue"
import Counter from "./Counter.vue"

test("increments on click", async () => {
  const { getByRole } = render(Counter)
  await getByRole("button").click()
  expect(screen.getByText("1")).toBeTruthy()
})
```

## What to skip

- Internal details (method names, private refs).
- Third-party libraries that are already tested.
- Giant snapshots nobody reads.

## Speed

Keep the suite fast and it becomes a habit. Slow, and it turns into a waiting line — then disappears.
