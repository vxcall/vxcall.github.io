---
title: Resurrecting flattened control flow from binary chaos
date: 2024-07-09 21:00:00 +0900
categories: [reverse engineering, deobfuscation]
tags: [asm, reverse-engineering, deobfuscation]
img_path: /assets/img/posts/resurrecting_flattened_control_flow_from_binary_chaos/
image:
  path: header.jpg
  lqip: /assets/img/posts/resurrecting_flattened_control_flow_from_binary_chaos/header.svg
  alt: header
---

## Introduction

This is a second article of manually deobfuscating binaries series! The obfuscation technique I stumbled upon today is called Control Flow Flattening.
Before describing anything, take a look at this flow graph.

IMAGE

## Table of Contents

- [Terms in this article](#terms-in-this-article)
- [[+]Control Flow Flattening in detail](#-control-flow-flattening-in-detail)
- [[+]a few potential approaches](#-a-few-potential-approaches)
- [[+] Annihilating dispatcher routines and restoring original flow](#statically-distinguish-original-basic-blocks)
- [statically distinguish original basic blocks](#use-symbolic-execution)
- [use symbolic execution](#-annihilating-dispatcher-routines-and-restoring-original-flow)
- [Conclusion](#conclusion)

## Terms in this article

The following are technical terms that appear frequently throughout this article

| term                         | short style      | description                                                                                 |
| :--------------------------- | :--------------- | :-------------------------------------------------------------------------------------------|
| Control Flow Flattening      | CFF              | name of an obfuscation technique                                                            |
| basic block                  | bb               | a set of assembly instructions semantically mashed into one                                 |
| original basic block         | obb              | a basic block which is from before obfuscation                                              |
| Breadth First Search         | BFS              | an algorithm for searching a tree data structure for a node that satisfies a given property |

## [+] Control Flow Flattening in detail

In a normal program, the execution flow typically follows a logical sequence, resulting in an assembly graph that progresses from top to bottom.
However, in a binary that has undergone control flow flattening (CFF), this pattern is significantly altered.

Control flow flattening is an obfuscation technique that 'flattens' the execution flow, making the graph horizontally long rather than vertically oriented.
This transformation aims to impede reverse engineers from easily tracking the original flow through static analysis.
To better illustrate this concept, consider the following simplified visual representation:

[IMAGE PLACEHOLDER]

During the obfuscation process, the CFF technique introduces a state machine into the binary. This state machine consists of two key components:

dispatcher
: This is the central control mechanism that manages the program's execution flow.

State variables
: These keep track of the current state of the program.

After applying CFF, all execution flows are controlled by the dispatcher based on the current state. The original sequence of instructions is broken down into basic blocks, each associated with a unique state.
The dispatcher determines which block to execute next by evaluating the current state, effectively obscuring the original control flow.

This flattened structure makes it challenging for reverse engineers to understand the program's logic and reconstruct its original flow.
Instead of following a clear, linear path, they must navigate through a maze-like structure where the relationships between code blocks are not immediately apparent.

## [+] a few potential approaches

#### statically distinguish original basic blocks

#### use symbolic execution

## [+] Annihilating dispatcher routines and restoring original flow

## Conclusion
![footer](footer.png)