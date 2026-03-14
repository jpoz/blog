---
title: "Rebuilding Excel's Formula Engine in a Weekend"
meta_title: ""
description: "How I orchestrated AI agents in parallel loops to build an XLSX calculation engine with 100+ formula functions in Go, from scratch, in a weekend."
date: 2026-03-13T05:00:00Z
image: "/images/posts/xlsx-ai.jpg"
categories: ["codez"]
authors: ["jpoz"]
tags: ["ai", "go", "excel"]
draft: false
---

Somewhere around 2 billion people use spreadsheets. Not "have access to", *use*. Over half of all businesses worldwide run on it. The average office worker spends [38% of their time](https://www.acuitytraining.co.uk/news-tips/new-excel-facts-statistics/) in a spreadsheet. Bloomberg called it a ["multitrillion-dollar empire."](https://www.bloomberg.com/features/2025-microsoft-excel-ai-software/) When people talk about the software that runs the global economy, they usually mean databases or ERP systems. They're wrong. It's spreadsheets. It's always been spreadsheets.

But what does it take to make `=SUMPRODUCT((A1:A100="Yes")*(B1:B100))` actually work? Building a proper calculation engine for spreadsheets (one that can actually evaluate formulas) has always been a massive undertaking. The spec is enormous, the edge cases are endless, and the function library alone runs to hundreds of entries. It's the kind of project that teams spend years on. Which is why almost nobody has done it well outside of Microsoft and Google. But AI changed the math on what's possible for a solo developer with a free weekend.

I built the base of [werkbook](https://github.com/jpoz/werkbook), a library that can read an XLSX file, evaluate its formulas, and write the results back... in a weekend. No shelling out to Excel or LibreOffice. It has a proper lexer, parser, bytecode compiler, virtual machine, hundreds of function implementations, and one weekend.

Here's how the weekend went.

## Friday Night: The Architecture

This was the part I did by hand. The formula engine is a classic compiler pipeline, and getting the architecture right matters more than anything downstream. AI is great at filling in patterns, but it needs the patterns to exist first.

I set up four stages:

1. **Lexer**: tokenize `SUM(A1:B12)` into meaningful pieces
2. **Parser**: handle operator precedence and turn tokens into an AST
3. **Bytecode Compiler**: compile the AST into stack-based VM instructions
4. **VM Evaluator**: execute the bytecode against actual cell data

I chose a bytecode VM over a tree-walking interpreter because spreadsheets get re-evaluated constantly. Compile once, run many times. With the skeleton in place and a handful of functions working end-to-end (basic arithmetic, `SUM`, `IF`), I had the scaffolding that AI could build on.

I also pointed AI at the XLSX file format and had it build out the read/write capability: parsing the XML inside the zip, resolving shared strings, handling styles. Tedious, spec-heavy work that AI handles well.

This is still the part of the job that's entirely human. The problem was loosely defined, "I want server-side spreadsheet calculation in Go," and it was on me to turn that into a specific architecture. What stages? What data structures? What tradeoffs? AI can't make those calls because it doesn't know what you're optimizing for.

## Saturday: The Loop

With the architecture solid and a few reference implementations to learn from, the next step was obvious: there are hundreds of Excel functions, and they mostly follow the same shape. Parse arguments, validate types, compute result, handle errors.

I set up an AI agent in a loop:

1. Find a formula function that isn't implemented yet
2. Implement it following the patterns established by the existing functions
3. Move on to the next one

This is where AI shines. Function #50 is structurally similar to function #5. The agent could look at how `AVERAGE` was implemented and produce `AVERAGEIF`. It could see how `LEFT` worked and build `RIGHT`, `MID`, `SUBSTITUTE`. The repetitive middle is AI's sweet spot.

By Saturday afternoon it was churning through math functions, statistical functions, text functions, date functions, logical functions, financial functions. The weirder corners of Excel too: Bessel functions, complex number arithmetic, the full `TEXT()` number formatting engine.

Before AI, this is the part that would have taken months. Not because any single function is hard, but because there are *so many of them*. The tedium is the bottleneck, not the complexity. AI removed the tedium entirely.

## Saturday Night: The Edge Finder

Raw implementations aren't enough. XLSX files have decades of quirky behavior baked in, and users expect exact compatibility. This is where my role shifted from "build" to "verify."

I spun up a second agent in parallel, focused entirely on breaking things.

This agent's job:

1. Review each implemented function
2. Think about edge cases — what happens with array inputs? Empty cells? Out-of-bounds references? Mixed types? Circular references? Boolean coercion? What about `#N/A` vs `#VALUE!` vs `#REF!`: does each function return the right error for the right reason?
3. Build failing test cases as actual `.xlsx` files

It found real problems. Functions that didn't handle arrays when they should. Off-by-one errors in date calculations. Missing coercion from strings to numbers. The kind of stuff that works fine in the happy path but falls apart the moment someone's spreadsheet does something slightly weird.

This is the part of the new job I find most interesting. I wasn't writing the tests myself. I was designing a system that would generate the right tests, deciding *what kinds of problems to look for* and letting the agent do the looking. The skill isn't writing the validation code. It's knowing what correct looks like.

With the edge finder producing a steady stream of failing test cases, I pointed a third agent at the failures:

1. Pick up a failing test
2. Figure out what's wrong
3. Fix it
4. Move on

Three agents, running in parallel. One building new functions. One finding problems. One fixing them. I'd check in periodically, review what they'd done, course-correct when something went sideways. But mostly they just... worked.

## Sunday: Real-World Validation

Unit tests and generated edge cases are one thing. Real spreadsheets from real people are another. On Sunday I went hunting for xlsx files in the wild: financial models, inventory trackers, project plans, anything I could find shared publicly online.

Here's the trick that makes this possible without even having Excel installed: xlsx files cache their calculated values. When someone saves a spreadsheet, Excel writes the last-computed result into every formula cell. That means you already have the answer key. Open the file, recalculate every formula with werkbook, and diff the results against the cached values. Any mismatch is a bug.

I pointed an agent at a pile of downloaded spreadsheets and had it run this comparison loop. It surfaced a handful of real issues: date serial number handling that didn't match Excel's 1900 leap year bug, locale-sensitive number formatting quirks, a few functions that silently dropped precision on large values. The kind of stuff you'd never catch with synthetic tests because you'd never think to write them.

## What I Ended Up With

By Sunday night: over 100 formula functions implemented, tested, and handling edge cases. Not a toy, something actually usable. You could hand it a real spreadsheet from a real business, change some input cells, recalculate, and get correct results back.

The conditional aggregates work (`SUMIFS`, `COUNTIFS` with wildcard matching). The lookup functions work (`VLOOKUP`, `INDEX`/`MATCH`). Dynamic array formulas work (`UNIQUE`, `FILTER`, `SORT` with spill ranges). Even the financial functions that nobody understands work.

## The Job is Different Now. And the Same.

The way I spent my time this weekend would look alien to me five years ago. I wrote maybe 20% of the code by lines. I spent most of my time designing systems (architecture, agent workflows, verification strategies) and then reviewing what those systems produced. My job was to know what "correct" looks like and build processes that could get there.

But here's what I didn't expect: the job felt familiar. A developer's job has always been to take a complicated, loosely defined problem and turn it into something very specific that actually works. That hasn't changed. What's changed is where you spend your time.

Zoom out, and the job description hasn't changed at all. Someone said "I need server-side spreadsheet calculation," a vague, complicated problem. I turned it into a specific, working solution. That's what developers do. That's what we've always done. We take an ambiguous need and make it concrete.

What's different is the leverage. The hard part of this project was never "how do I implement `AVERAGEIF`." It was "how do I structure this so that hundreds of functions can be implemented correctly, and how do I know they're correct?" The architecture. The verification system. The taste for what matters.

AI didn't change the job. It changed where the bottleneck is. It used to be writing the code. Now it's knowing what to build and knowing when it's right. The developers who thrive with these tools won't be the ones who write the most code. They'll be the ones who can look at a messy, underspecified problem and see the system that solves it.

That part is still entirely human. And I don't think that's changing any time soon.
