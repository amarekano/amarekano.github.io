---
layout: post
title:  "WebKit JavaScriptCore Internals"
date:   2021-05-26 19:00:00 +0000
categories: jekyll update
---

I've been putting together a series of blog posts on deep diving into the internals of JavaScriptCore, which is the JS engine that powers WebKit. This series was created to help me with crash analysis and bug triage when fuzzing JavaScriptCore with Fuzzilli. The blog series is hosted at [zon8.re](https://zon8.re/posts/).

The series covers the four execution tiers in the engine and walks through the process of converting javascript into machine code that is exeucted by the browser.

[Part I: Tracing JavaScript Source to Bytecode](https://zon8.re/posts/jsc-internals-part1-tracing-js-source-to-bytecode/)

[Part II: The LLInt and Baseline JIT](https://zon8.re/posts/jsc-internals-part2-the-llint-and-baseline-jit/)

[Part III: The DFG (Data Flow Graph) JIT – Graph Building](https://zon8.re/posts/jsc-part3-the-dfg-jit-graph-building/)

[Part IV: The DFG (Data Flow Graph) JIT – Graph Optimisation](https://zon8.re/posts/jsc-part4-the-dfg-jit-graph-optimisation/)

[Part V: The DFG (Data Flow Graph) JIT – On Stack Replacement](https://zon8.re/posts/jsc-part5-the-dfg-jit-osr/)

Parts to VI and VII are work in progress and cover the FTL JIT engine. Once published on zon8, this post will be updated.