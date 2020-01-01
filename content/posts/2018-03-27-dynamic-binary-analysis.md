---
layout: post
title:  "Getting started with Dynamic Binary Analysis"
comment: true
aliases: [/dynamic-binary-analysis/]
date:   2018-03-27 18:18:00
tags:
- linux
- reverse
- binary analysis
description: ''
images:
- author.jpg
categories:
- Reverse Engineering 
---

![image](https://memegenerator.net/img/instances/58266813/everybodys-out-partying-and-im-sitting-here-reverse-engineering.jpg)

# What, Why, and Where?

## What is it exactly?

Dynamic Binary Analysis (DBA) is a technique to analyze the behavior of a binary by somehow running it and watch its behavior. Obviously it's the opposite of Static Binary Analysis in which you disassemble a piece of code and draw the graph of the entire program to see what it does. 

## Why not Static?

well, it's actually better if you have a small binary or a binary written in a friendly programming language. But when your binary gets big enough, or you use C++, Rust or Go as your language of choice, it becomes increasingly difficult to see what the binary is doing. 

# OK let's go through it

## DBA or DBI?

As I mentioned, DBA is a technique. For example, there are some DBA tools to profile your memory and visualize it so you can go back and forth with your execution and see the memory mapping in each step. As you can already imagine, this is a very useful way of analyzing the behavior of a binary and see exactly what's going on inside it without reading so much assembly.

But DBA itself relies on a framework that makes it possible. Those underlying framework are called DBI or Dynamic Binary Instrumentation

## Instrumentation is cool

Dynamic Binary Instrumentaion is a technique to analyze and modify the behavior of a binary program by injecting arbitrary code at arbitrary places while it is executing.

It basically gives you an API to "hook" wherever you want in the binary while it's running. Then you can do whatever you want with the binary at that moment. One of my favorite tools of DBI is Intel's Pin. Its API allows injecting C/C++ arbitrary code. Cool, right?

Alternatives to Pin:

+ DynamoRIO (Win, Linux, Mac)

[DynamoRIO](http://www.dynamorio.org/) is a runtime code manipulation system that supports code transformations on any part of a program, while it executes. DynamoRIO exports an interface for building dynamic tools for a wide variety of uses: program analysis and understanding, profiling, instrumentation, optimization, translation, etc. Unlike many dynamic tool systems, DynamoRIO is not limited to insertion of callouts/trampolines and allows arbitrary modifications to application instructions via a powerful IA-32/AMD64/ARM/AArch64 instruction manipulation library. DynamoRIO provides efficient, transparent, and comprehensive manipulation of unmodified applications running on stock operating systems (Windows, Linux, or Android) and commodity IA-32, AMD64, ARM, and AArch64 hardware. Mac OSX support is in progress.

+ Valgrind (Linux)

[Valgrind](http://valgrind.org/) is an instrumentation framework for building dynamic analysis tools. There are Valgrind tools that can automatically detect many memory management and threading bugs, and profile your programs in detail. You can also use Valgrind to build new tools.

### Pin

Pin provides a rich API that abstracts away the underlying instruction-set and allows context information such as register contents to be passed to the injected code as parameters. Pin automatically saves and restores the registers that are overwritten by the injected code so the application continues to work. Limited access to symbol and debug information is available as well. The Pin framework download comes with a set of pre-created tools called ‘Pintools’.

pin in windows:

```
pin.bat -t pintool.dll [pintool args] -- program.exe [program args]
pin.bat -pid <program pid> -t pintool.dll [pintool args]
```

Pin works in a very sophisticated way. The description in the Pin manuals to think of Pin as a JIT (just in time) compiler, where the compiler does not take byte code (as JIT compilation does with Java), but the executable of the process pin is executed against. This means pin inserts itself into the process’ execution. This can be seen when looking at the memory map of such a process:


### pinatrace

One of Pin's biggest tools is its Memory Reference Trace. The pinatrace tool basically generates a file with every single memory access of a process. This allows you to get an understanding what happens within a function. This means you can determine what information or data is accessed in what function


# Triton

We've talked enough about DBI and their entire ecosystem. Now let's talk about the tools built with them. I picked my favorite one to talk about 

+ Triton (Linux, Windows, Mac)

[Triton](http://triton.quarkslab.com/) is a Dynamic Binary Analysis (DBA) framework. It provides internal components like a Dynamic Symbolic Execution (DSE) engine, a Taint Engine, AST representations of the x86 and the x86-64 instructions set semantics, SMT simplification passes, an SMT Solver Interface and, the last but not least, Python bindings.

![image](https://camo.githubusercontent.com/d06dfe7f98f26fcddad3d29c7e65eab363d60bd2/687474703a2f2f747269746f6e2e717561726b736c61622e636f6d2f66696c65732f747269746f6e5f7630335f6172636869746563747572652e737667)


Based on these components, you are able to build program analysis tools, automate reverse engineering and perform software verification

As you can see, Triton supports all my mentioned DBIs and then some! It has provided a great and unified API to deal with Taint Analysis, and 'Symbolic Execution'. I think Dynamic Symbolic execution needs its own blog post.

Note: This post will be edited soon because it's a very big topic :-)