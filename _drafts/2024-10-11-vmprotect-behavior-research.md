---
title: Let's Get Rekt by a VMProtected Binary with 100% Complexity + 10VMs
date: 2024-10-10 16:00:00 +0900
categories: [reverse engineering, virtualization]
tags: [asm, reverse-engineering, virtualization]
img_path: /assets/img/posts/vmprotect-behavior-research/
image:
  path: header.jpg
  lqip: /assets/img/posts/vmprotect-behavior-research/header.svg
  alt: header
---

## Introduction

Virtualization is one of the most hard-to-break protection techniques employed widely by both legitimate and malicious applications. I'm totally hooked on it because knowing vmprotect is getting paramount in 2024 to reverse engineer softwares with good security like anti-cheat system.

A few weeks ago a friend of mine gave me a binary virtualized with latest VMProtect3.8.1 with 100% complexity + 10vm. Although I'm an absolute newbie in this field, and had no idea how the hell VMProtect works, I did a bit of research about its behavior so I'm writing a quick note about it.

> This article content might be too rudimentary for those who ever dealt with VMProtect before.
{: .prompt-info }

## About virtualization

Virtualization is by far the strongest obfuscation done by transforming legit code into a custom bytecode that runs on a virtual machine (VM) embedded within the protected application. The virtual machine run in the application typically has a proprietary architecture that the decompiler like IDA doesn't recognize the bytecode(original code) as meaningful instructions and ended up with just showing a chunk of pointless-looking bytes. I'll show you the actual image of IDA's disassemble view of it later. Among couple of softwares that have virtualization capabilities, I'll take a look at a binary virtualized with VMProtect version 3.8.1.

VMProtect is an infamous software known for its strong protection techniques being developed by Russian firm. It applies not only virtualization to software but other obfuscation techniques such as deadstores insertion, opaque predicates, indirect jmp, stack manipulation...you name it!

Anyways, following is a figure showing the brief overview of VMProtect.

![vmprotect overview](vmprotect_overview.png)
_VMProtect overview_

So it eats the original binary and spits out a new binary with a VM embedded in it. VMProtect has been updated frequently and considering the latest version as of now is 3.9, version 3.8.1 (the one my binary was virtualized with) is pretty new I'd say.

So this is what you should know before diving into the actual contents!

## The binary's behavior

## Let's get rect!

Upon writing this post, I had read a couple of papers and articles , so I had a general idea of where to start.

## Conclusion

![footer](footer.jpg)
