---
title: Let's Get Rekt by a VMProtected Binary with 100% Complexity
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

Virtualization is one of the most hard-to-break protection techniques employed widely by both legitimate and malicious applications.
And I'm totally hooked on it because knowing vmprotect is getting paramount in 2024 to reverse engineer softwares with robust security like anti-cheat software.

A couple of weeks ago, a friend of mine gave me a binary virtualized with VMProtect3.8.1 with 100% complexity + 10vms.
Although I'm an absolute newbie in this field, I always wanted to know about virtualization technique I finally decided to tackle on it and write a diary about how I get wrecked! lol

In this blog post I'll write about what I saw and felt during analysis, it's neither something covers entire VMProtect's feature nor in-depth, but more like initial brief research. Cuz that'd be beyond my wheelhouse.

> This article might be too rudimentary for those who ever dealt with VMProtect before. Also there might be misunderstandings, forgive me I'm not a professional.
{: .prompt-warning }

## About virtualization

Virtualization is by far the strongest obfuscation done by transforming legit code into a custom bytecode that runs on a virtual machine (VM) embedded within the protected application.
The virtual machine that runs in the application typically has a proprietary architecture that the static disassembler like IDA doesn't recognize the bytecode(original code) as meaningful instructions and ended up with just showing chunks of bytes.
I'll show you the actual image of IDA's disassemble view of it later.

Among couple of softwares that have virtualization capabilities, this time I'll take a look at a binary compromised with VMProtect version 3.8.1.
VMProtect is an infamous software known for its strong protection techniques being developed by Russian firm, VMProtect Software.
It applies not only virtualization to software but other obfuscation techniques such as deadstores insertion, opaque predicates, indirect jmp, stack manipulation...you name it!

VMProtect is does bin2bin[^bin2bin] obfuscation. it eats the original binary and spits out a new binary with a VM embedded in it.
VMProtect has been updated frequently and considering the latest version as of now is 3.9, version 3.8.1 (the one my binary was virtualized with) is pretty new I'd say.

Following figure is showing the brief overview of VMProtect.

![vmprotect overview](vmprotect_overview.png)
_VMProtect overview_

So this is what you should know before diving into the actual contents!

> Btw, I'll call VMProtect, 'vmp' from now on!
{: .prompt-info }

## The binary's behavior

Knowing what the binary does is the most important thing to know when you reverse enginer software.
So that you can presume for instance what kind of Windows API you would expect to be called or where to start.
This particular one is a very simple vmp sample, so it was as straightforward as just print out number `2` and pop up a messagebox with a word "wait".

![binary_behavior](binary_behavior.png)
_it's print out '2' and pop up a message box_

## Let's get rect!

Upon writing this post, I had read a couple of papers and articles, also I peeked (wouldn't say analyse tho lol) at VMProtect2 binary a bit before so I had a general idea of where to start.
Moreover my friend told me where the vm begins, people call it vm_entry, so so that will also be a starting point.

But still, the binary is obfuscated with 100% complexity, it could be more and more complicated than what I expect of course.
Don't worry it's gonna be totally fine, cuz we're here to get wrecked!

## [+] Disable ASLR

I noticed that the binary was built with ASLR[^aslr] on, which makes ma unpleasant.
So I first turned it off.

It's pretty easy, open it with your favorite hex editor and navigate to `NTHeader` -> `OptionalHeader` -> `DllCharacteristics` -> `IMAGE_DLLCHARACTERISTICS_DYNAMIC_BASE` and set it to 0.

Then go to Windows security in windows settings and in `App & browser control` tab, turn off `Randomize memory allocations (Bottom up ASLR)`.

That way you can turn off the ASLR and fix the image base across reboot I believe.

> also setting `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\kernel\MitigationOptions`'s third significant bit to 0 does the same thing in registry.
{: .prompt-tip }

## [+] Find vm entry indication

Right off the bat, look at the main function.

![main](main.png)
_main function_

Right before going into vm_entry it's assigning value to the 3 registers.
So far you can assume that they're used in vm_entry maybe? Keep it in mind anyway and carry on.

```nasm
mov     r8d, 3
mov     edx, 2
mov     ecx, 1
```

Now let's look at the vm_entry (image below).
As you can see it's pushing 2 registers before going into a subroutine which I remember it was decrypted VIP (like RIP but in VM context) in VMProtect2, but not sure this time `r14` is 0 and `r9` is holding a stack memory address that was initialized in `_initterm` function.
No idea if it's related to vmp or not.

![vm_entry](vm_entry.png)
_vm entry_

Btw In the nature of vm_entry, it should have a specific initialization process.
Since it's going to use general registers in its vm, it first push all the registers to stack to preserve the current state for later.
However, based on my memories of VMProtect2, I overconfidently thought 'This should be easy' and ended up learning a painful lesson.

In vmp3, the contents of vm_entry were finely chopped up, and it seemed that a single operation was performed through multiple functions.
In the VMP2 I had seen before, all register pushes were done in a single function, but this time even the pushes were divided into multiple functions and basic blocks, making it very difficult to search through

I managed to find it, as I said it was chopped up, but the image below is the biggest piece of function which pushes many general registers.
The function has a branch in the middle of it however it turned out to be a part of the obfuscation, Opaque predicates[^opaque_predicates].
I commented in the picture why it'd always takes same branch.

![vm_entry2](vm_entry2.png)
_red: opaque predicates, green: register push_

## [+] Locate VIP initialization

Many instructions in virtualized subroutine doesn't make sense to me and it confuses me at the same time. Idk what to look for and shit.
However 1 thing I know was vmp must've stored custom bytecode for vm somewhere in the binary.
And wherever it is, the virtualized process will eventually have to access it.

So I decided to debug the program and closely monitor the registers and stack to see if any of them started holding the bytecode pointer, usually refered to as `VIP` (virtual instruction pointer).
It felt like it took ages to find it, but I finally discovered! In the image below, `R9` contains `0x100000000`, and `R10` holds the encrypted `VIP`.
`R10` has been decrypted along the way, and by combining it with `R9`, the decryption process has been completed.

```
0x7FF5A9EC114D + 0x100000000 = 0x7FF6A9EC114D
```

![bytecode ptr](bytecode_ptr.png)
_virtual instruction pointer is being decrypted_

And following is the where the ptr points to.

![bytecode](bytecode.png)
_this is what vmprotect custom bytecodes look like_

To ensure it's correct one, I kept going and look for instructions that uses `R10`. Then I found the part where it retrieve a byte from VIP address-7 and moving VIP forward. 

![vip ref](vip_ref.png)
_getting a byte from vip and updating pointer_

vmp stores opcode and oprand in bytecode field. So when it wants to execute `add eax, 5`, retrieve each opcode and operand just like I showed you, and execute it.

> Note that VIP looks technically going backward just like stack does, but apparently it varies depending on the vm you're dealing with. Some vms actually go forward and others go backward.
{: .prompt-tip }

## [+] Virtual stack initialization

![virtual stack](virtual_stack.png)

## [+] Deadstore removal plugin

The amount of deadstore vmp inserts between legit instructions are insane that I almost lose my temper so I quickly made an IDA plugin to remove all the deadstores in the current function. 

[vxcall/deadstore-remover](https://gist.github.com/vxcall/1b2841370d07f25dc0d729985306bf4f)

Place it in IDA's `plugins` folder.
It's utilizing a library called [triton](https://github.com/jonathansalwan/Triton), therefore you have to build it to use my plugin.
Also it works on IDA 8.X but doesnt work on IDA 9.X due to the API compatibility.
You can use it by placing cursor on the middle of a function in IDA and run the plugin I named it "Function Deadstore remover", so that the plugin will analyze and nop out every garbage instructions.

![deadstore-remover](deadstore_remover.png)
_nop out deadstore, if the instruction made out of multipul bytes, it truncates them_

## [+] Devirtualize it

Enough research, it's time to devirtualize it.
For it I used [NaC-L/Mergen](https://github.com/NaC-L/Mergen) which is a very cool tool to lift obfuscated binary into LLVM IR[^llvm_ir]. It gets assembly as input and interprets each disassembly's semantics into LLVM IR using LLVM/IRBuilder. Moreover, it utilizes LLVM's optimization feature to deobfuscate/devirtualize code.
That way Mergen can universary handle not only VMProtect but other obfuscators and virtualizers.
I found it so cool that I made a couple of small pull requests to contribute it! You have to check it out!

So this is the output when I run Mergen on the binary.

![mergen_output](mergen_output.png)
_the output of mergen_

Following is the lifted LLVM IR

```llvm
; ModuleID = 'my_lifting_module'
source_filename = "my_lifting_module"

; Function Attrs: mustprogress nofree norecurse nosync nounwind willreturn memory(argmem: write)
define i64 @main(i64 %rax, i64 %rcx, i64 %rdx, i64 %rbx, i64 %rsp, i64 %rbp, i64 %rsi, i64 %rdi, i64 %r8, i64 %r9, i64 %r10, i64 %r11, i64 %r12, i64 %r13, i64 %r14, i64 %r15, ptr nocapture readnone %TEB, ptr nocapture writeonly %memory) local_unnamed_addr #0 {
entry:
  %GEPSTORE-5368971276-4154 = inttoptr i64 1376064 to ptr
  store i64 %r8, ptr %GEPSTORE-5368971276-4154, align 4
  %GEPSTORE-5368930894-4155 = inttoptr i64 1376056 to ptr
  store i64 %rdx, ptr %GEPSTORE-5368930894-4155, align 4
  %GEPSTORE-5369500900-4156 = inttoptr i64 1376048 to ptr
  store i64 %rcx, ptr %GEPSTORE-5369500900-4156, align 4
  %realnot-5369329545- = sub i64 %rcx, %rdx
  %adc-temp-5369316182- = add i64 %realnot-5369329545-, %r8
  ret i64 %adc-temp-5369316182-
}

attributes #0 = { mustprogress nofree norecurse nosync nounwind willreturn memory(argmem: write) }
```
{: file='output.llvm'}

As you can see it's basically doing `rcx - rdx + r8` and returning the result, which every register looks familiar to me.
Yes they're the ones initialized before going into vm_entry.
They are

- `RCX`: 1
- `RDX`: 2
- `R8`: 3

respectively.

So the final equation will be `1 - 2 + 3 = 2`
vm_entry turned out to be returning 2.

## Conclusion

![footer](footer.jpg)

After everything I've been through in this article, I still feel VMProtect is too overwhelming for me but at least gained a lot of fundamental knowledges about it in action and I'm totally satisfied with it!


## Footnotes

[^bin2bin]: **bin2bin**: a type of obfuscation software. stands for binary to binary. There's another type linker level obfuscation according to [es3n1n's Blog](https://blog.es3n1n.eu/posts/obfuscator-pt-1/).
[^aslr]: **ASLR**: Address Space Layout Randomization. When it is enabled, the binary will be loaded at random location each time which means you have to rebase the program in IDA everytime open it with x64dbg.
[^opaque_predicates]: **Opaque predicates**: a type of obfuscation techniques which generates a seemingly legit branches that is actually a garbage branching taking you the same branch every time. 
[^llvm_ir]: **LLVM IR**: LLVM's intermediate representation.