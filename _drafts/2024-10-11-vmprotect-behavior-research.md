---
title: Brief Research of Virtualized Binary with VMProtect with 100% Complexity
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

> This article content might be too rudimentary for you if you are a skilled reverse engineer.
{: .prompt-info }

## About virtualization

Virtualization is a unique method used to obfuscate code by transforming it into a custom bytecode that runs on a virtual machine (VM) embedded within the protected application. The virtual machine run in the application always has a proprietary architecture so the decompiler like IDA doesn't recognize the bytecode(original code) as neither instruction nor function but instead just a chunk of bytes.

## What is VMProtect?

VMProtect is one of the most famous (and infamous) protection software being developed by Russian firm, VMProtect Software. What it applies to software is not only virtualization but other obfuscation such as deadstores insertion, opaque predicates, indirect jmp, stack manipulation...you name it!

VMProtect has been updated frequently and considering the latest version as of now is 3.9, version 3.8.1 (the one my binary was virtualized with) is pretty new I'd say.
