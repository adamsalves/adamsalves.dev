+++
title = "Contract before code: how I run AI on my projects"
date = 2026-08-27
draft = false
description = "The techniques I actually use day to day with AI — context as a budget, retrieval before generation, subagents with clean context, and a loop with a verification step — and the numbers they produced across 7,905 commands."
tags = ["ai", "rag", "agents", "workflow"]
toc = true
+++

Almost everything written about working with AI is about prompting. In practice the prompt is the smallest decision in the room. What decides the outcome is what enters the context, in what order, who checks the output, and how many times the cycle runs before anyone accepts the work.

This post is about the techniques that survived a year of doing this every day, with the numbers they produced.

## Contract before code

The first technique isn't technical at all: write the decision down before asking for execution. A short document with two sections doing the heavy lifting — "Closed Decisions" and "Out of Scope" — and nothing starts before it exists.

That solves a problem specific to working with an agent. A model is extraordinarily good at building what you asked for, including when what you asked for is wrong, and it will not stop halfway to ask whether the premise holds. Writing the decision first is the only moment when changing your mind still costs one line of text.

"Out of Scope" carries more weight than it looks. Without it, an agent with context to spare finds adjacent work to do, and the diff grows in directions nobody asked for.

## Retrieval before generation

Grep is string search. Ask it "who calls this function" and it hands back every line containing the name: the definition, the comments, the log string, the test that mentions it in a title — and the model reads all of that in order to throw nearly all of it away.

Instead, the repositories I work in carry a graph of their own code, built with Tree-sitter and queried before any file gets read. The largest one holds 8.5 MB of indexed nodes and edges. The questions I ask it aren't textual, they're structural — who calls this, what does this import, which tests cover it, what's the blast radius of changing it.

This is RAG in the sense that matters: retrieve the right piece before generating, instead of dumping the repository into the context and hoping for attention. The difference from document RAG is that the index here isn't an embedding of loose text — it's the actual structure of the code, so "who calls this function" has an exact answer rather than a similarity approximation. Semantic search sits on top, for when I don't know the name of what I'm after.

The practical effect: a question that used to cost reading six files now costs one query and a few hundred tokens of answer.

## Where tokens actually leak

Every shell operation I run goes through a proxy that filters the output before it reaches the context. Across 7,905 commands it has cut **3.0 million tokens, 47.3% of the total**.

The interesting part isn't the average, it's the distribution:

```
test runner              123 runs    98.2%
gh pr diff                73 runs    24.5%
grep                     511 runs    10.3%
file read              1,240 runs     8.3%
```

One hundred and twenty-three test runs saved 1.6 million tokens on their own, more than half the total. Reading files, done ten times as often, saved 250 thousand.

The lesson is in the gap between 98.2% and 8.3%. A test runner's output is progress bars, per-file timing, repeated stack traces and a summary — nearly all of it formatting, and the signal fits in three lines. Source code is almost all signal, and compressing that costs understanding. Context optimization that pays off lives in tool noise, not in the material you want the model to understand.

## Subagents are isolation, not parallelism

The "agentic" part of my work is less exciting than the term suggests. It isn't a fleet of agents cooperating. It's two things.

The first is dispatching each task to a subagent with its own context, so the debris of one doesn't contaminate the next. The second is the one that matters: **code review runs in a subagent that never watched the code being written.**

That isn't ceremony. A model reviewing its own work carries in context the reasoning that produced the code, and that reasoning is exactly what needs questioning. It rereads and agrees with itself. With a clean context, the same model reads the diff as a stranger's code and returns an explicit verdict — "changes required" or "approved with suggestions". That's how a cross-tenant authorization flaw and a logged credential surfaced, both of them missed by the session that wrote the code.

The final gate isn't an agent at all: never commit straight to main, every change through a PR. The AI proposes and does not publish.

## The loop, and the budget written inside it

Loop engineering, in practice, is deciding where the cycle stops. Mine is a checkbox plan, one task at a time, each in TDD with the red and the green recorded, and an explicit check before the commit.

That check is the step almost everyone skips, and it's the only thing keeping the loop from converging on "the agent said it passed". Two of my checks cost one incident each: `ls` the directory against the plan's file list, because a subagent creates files outside the convention and a diff won't show what you never asked for; and reproduce the CI environment before calling anything green, because the local test runner reads an environment variable CI doesn't have.

What I didn't expect is that the context budget had to be written inside the loop itself. The skills I carry into every project end with a literal block:

> Target: complete any review/debug/refactor task in ≤5 tool calls and ≤800 total output tokens.

Without a stated ceiling, an agent with good tools uses all of them. The limit doesn't make the answer worse, it makes the search deliberate — and the instruction just above it, "always start with `get_minimal_context`", is what makes it fit.

There's a slower loop wrapped around all of them: every correction I give becomes a memory and applies in later sessions. There are 159 of them now. The gain isn't the model learning — it's me not typing the same correction a fourth time.

## The boundary between instruction and data

One thing I treat as a requirement rather than extra care: user input is never interpolated into a prompt. The `user` message stays separate from the `system` one.

Concatenating the two is the same class of mistake as building SQL with `+`, and it has the same fix: the boundary between instruction and data has to exist in the format, not in the hope that the data behaves. Any model reading a tool result, a web page or a third-party file is one concatenation away from doing what that text says.

## Where this touches the work

None of these techniques is about writing code faster. They're about the one thing that gets expensive once code starts arriving fast: deciding what to accept.

A contract before execution decides what's worth building. Structured retrieval and output filtering decide what the model sees. Clean context at review and a human gate at merge decide what gets in. The loop with a verification step decides when to stop.

The cost is real and worth stating: writing the decision first takes time that a small task won't repay, and I don't do it there. On a large change it comes out cheaper than finding out halfway through.
