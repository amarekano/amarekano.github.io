---
layout: post
title:  "A Short Survey on JavaScript Fuzzers"
date:   2020-02-04 19:00:00 +0000
categories: jekyll update
---

## Introduction

I've spent some time in 2019 getting started with browser exploitation specifically hunting for bugs in JavaScript engines. There's typically two ways to approach bug hunting for vulnerabilities: source code review and fuzzing. I am fairly comfortable developing in JavaScript but the internals of a JavaScript engine were unfamiliar to me. Diving into source code review on a massive and unfamiliar codebase seemed like an onerous task for someone just getting started with browser exploitation. 

A relatively cheaper and attractive alternative (read lazy option) was to build a fuzzer that automates bug hunting. Fuzzers are generally of two types, mutation based (typically used to find bugs in binary file formats or binary protocols) and generation based (typically used to find bugs in software where the input is created by a grammar specification. For example generating valid JavaScript programs). 

Once a suitable target has been identified, the process of fuzzing can be summarised in three main steps:
1. Identify the input points to the application (i.e. binary data or structured data)
2. Generate random payloads that are syntactically and/or semantically valid for the input points.
3. Submit the generated payload to the input point and observe program behaviour. 

Should application crash, triage the crash for exploitability. Repeat steps 2 and 3 to collect multiple crash dumps/logs for triage. Obviously, this is a very simplistic view of fuzzing and this post is neither a primer on fuzzing JavaScript engines nor about exploiting engine bugs. More post to follow on that ;)

This post attempts to document my experience building and using some publicly available fuzzers and hopefully be useful to anyone looking to get started with JavaScript engine fuzzing. 

## What makes for an efficient fuzzer?

I found this useful [blogpost](https://chromium.googlesource.com/chromium/src/testing/libfuzzer/+/f44bd1a3e684b72de7a90871a51ab4acb0df4054/efficient_fuzzer.md) from the fine people at Chromium, that describes the key characteristics of an efficient fuzzer. These are summarised as follows: Speed at which payloads are generated and executed, An extensive grammar library and/or corpus of payloads to initialise the fuzzer and code coverage that the fuzzer achieves. 
	
## Dharma + libfuzzer
I first heard of dharma from reading [ret2's pwn2own blog series](https://blog.ret2.io/2018/06/05/pwn2own-2018-exploit-development/). Ret2 reconstructed JavaScript grammar using dharma and wrapped the generator into a fuzzing harness to fuzz webkit. Their dharma grammars for JavaScript or their fuzzing harness aren't publicly available but they did document their methodology on how to design a payload generator using dharma.

Using this information, I hacked together dharma and libfuzzer to create [dharmafuzz](https://github.com/amarekano/dharmafuzz). This was a very primitive fuzzer to fuzz v8. The fuzzer uses is a python bridge between Dharma and libfuzzer and relies on manually generated JS grammars to construct fuzzing payloads. The fuzzer is primitive since it does not take into account coverage-guided fuzzing, corpus generation/seeding or use weighted grammars.

## jsFunfuzz + FuzzManager
Manually recreating JavaScript grammar in dharma was time consuming and labour intensive. Looking through some of Mozilla's github repositories, I found [jsFunfuzz](https://github.com/MozillaSecurity/funfuzz/tree/master/src/funfuzz/js/jsfunfuzz). This is significantly better than dharmafuzz since its developers have already covered majority of the JS specification and thus negates the need to recreate JavaScript grammars from scratch. Jsfunfuzz also introduces weighted grammars which allowed tweaking the fuzzer to generate payloads of a certain configuration with higher probability. jsFunFuzz, however, does not generate a corpus or implement coverage-guided fuzzing. With some minor modifications, I've managed to patch jsFunFuzz to fuzz v8 and this patched version can be cloned from the following github [repo](https://github.com/amarekano/v8-jsfunfuzz). 

## Fuzzilli
This is possibly the first coverage guided JavaScript fuzzer developed by [saelo](https://twitter.com/5aelo) and was publicly released in March 2019. [Fuzzilli](https://github.com/googleprojectzero/fuzzilli) is JS engine agnostic currently support v8, JavaScriptCore and SpiderMonkey. Fuzzilli was a massive step-up from dharmafuzz and jsFunFuzz. It offers coverage-guided fuzzing, corpus generation/seeding and weighted grammars. I highly recommend reading [Saelo's thesis](https://saelo.github.io/papers/thesis.pdf) on coverage-guided fuzzing for JavaScript engines.
