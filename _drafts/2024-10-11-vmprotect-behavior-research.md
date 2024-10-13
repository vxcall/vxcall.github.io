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

Virtualization is one of the most hard-to-break protection techniques employed widely by both legitimate and malicious applications. And I'm totally hooked on it because knowing vmprotect is getting paramount in 2024 to reverse engineer softwares with good security like anti-cheat system.
After everything I've been through in this article, I still feel VMProtect is too overwhelming for me but at least gained some fundamental knowledge about it and I'm satisfied.

A few weeks ago a friend of mine gave me a binary virtualized with almost latest VMProtect3.8.1 with 100% complexity + 10vm. Although I'm an absolute newbie in this field, I always wanted to know about virtualization I finally decided to work on it and write a diary about how I get rekt!

> This article content might be too rudimentary for those who ever dealt with VMProtect before.
{: .prompt-warning }

## About virtualization

Virtualization is by far the strongest obfuscation done by transforming legit code into a custom bytecode that runs on a virtual machine (VM) embedded within the protected application. The virtual machine run in the application typically has a proprietary architecture that the decompiler like IDA doesn't recognize the bytecode(original code) as meaningful instructions and ended up with just showing a chunk of pointless-looking bytes. I'll show you the actual image of IDA's disassemble view of it later. Among couple of softwares that have virtualization capabilities, I'll take a look at a binary virtualized with VMProtect version 3.8.1.

VMProtect is an infamous software known for its strong protection techniques being developed by Russian firm. It applies not only virtualization to software but other obfuscation techniques such as deadstores insertion, opaque predicates, indirect jmp, stack manipulation...you name it!

Anyways, following is a figure showing the brief overview of VMProtect.

![vmprotect overview](vmprotect_overview.png)
_VMProtect overview_

So it eats the original binary and spits out a new binary with a VM embedded in it. VMProtect has been updated frequently and considering the latest version as of now is 3.9, version 3.8.1 (the one my binary was virtualized with) is pretty new I'd say.

So this is what you should know before diving into the actual contents!

> Btw, I'll call VMProtect, 'vmp' from now on!
{: .prompt-info }

## The binary's behavior

Knowing what the binary does is the most important thing to know when you reverse enginer software.
This particular one is a vmp test sample, so it was as simple as just print out number `2` and pop up messagebox with a word "wait".

![binary_behavior](binary_behavior.png)

## Let's get rect!

Upon writing this post, I had read a couple of papers and articles, also I peeked (wouldn't say analyse tho lol) at VMProtect2 binary a bit before so I had a general idea of where to start. Moreover my friend told me where the vm begins, people call it vm_entry, so so that will also be a starting point.

But still, the binary is obfuscated with 100% complexity, it could be more and more complicated than what I expect of course.
Don't worry it's gonna be totally fine, cuz we're here to get wrecked!

## vm_entry

So the image below is the vmentry. As you can see it's pushing 2 registers before going into a subroutine which I remember it was decrypted VIP (like RIP but in VM context) in VMProtect2 but not sure, this time r14 is 0 and r9 is some stack memory address that was initialized in `_initterm` function. idk if it's related to vmp or not.

![vm_entry](vm_entry.png)

In the nature of vm_entry, it should have a specific initialization process. Since it's going to use general registers in its vm, it first push all the registers to stack to preserve the current state for later. However, based on my memories of VMProtect2, I overconfidently thought 'This should be easy' and ended up learning a painful lesson.
In vmp3, the contents of vm_entry were finely chopped up, and it seemed that a single operation was performed through multiple functions. In the VMP2 I had seen before, all register pushes were done in a single function, but this time even the pushes were divided into multiple functions and basic blocks, making it very difficult to search through

I managed to find it, as I said it was chopped up, but the image below is the biggest piece of function which pushes many general registers.
The function has a branch in the middle of it however it turned out to be a part of the obfuscation, Opaque predicates. I commented in the picture why it's always taking same branch.

![vm_entry2](vm_entry2.png)

## Locate VIP initialization

I decided to debug the program and closely monitor the registers and stack to see if any of them started holding the bytecode pointer usually refered to as VIP (virtual instruction pointer). It felt like it took ages to find it, but I finally succeeded! In the image below, `R9` contains `0x100000000`, and `R10` holds the encrypted VIP. `R10` has been decrypted along the way, and by combining it with `R9`, the decryption process has been completed.

```nasm
0x7FF5A9EC114D + 0x100000000 = 0x7FF6A9EC114D
```

![bytecode_ptr](bytecode_ptr.png)

And following is the where the ptr points to.

![bytecode](bytecode.png)

To ensure it's correct one, I kept going and look for instructions that uses `R10`. Then I found the part where it retrieve a byte from VIP address-7 and moving VIP forward. 

![bytecode ptr ref](bytecode_ptr_ref.png)

vmp stores opcode and oprand in bytecode field. So when it wants to execute `add eax, 5`, retrieve each opcode and operand just like I showed you, and execute it.

> Note that VIP looks technically going backward just like stack, but apparently it varies depends on the vm you're looking at. some vm actually go forward and others go backward.
{: .prompt-tip }

## deadstore removal plugin

The amount of deadstore vmp inserts between legit instructions are insane that I almost lose my temper so I quickly made an IDA plugin to remove all the deadstores in the current function. 

[vxcall/deadstore-remover](https://gist.github.com/vxcall/1b2841370d07f25dc0d729985306bf4f)

Place it in IDA's plugins folder. It's utilizing a library called [triton](https://github.com/jonathansalwan/Triton), you have to build it in order to use my plugin. Also it works on IDA 8 but doesnt work on IDA 9 due to API compatibility.
You can use it by placing cursor on the middle of a function in IDA and run the plugin I named it "Function Deadstore remover", so that the plugin will analyze and nop out every garbage instructions.

## Conclusion

![footer](footer.jpg)
