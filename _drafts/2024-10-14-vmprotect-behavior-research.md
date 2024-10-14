---
title: Let's Get Rekt by a VMProtected Binary with 100% Complexity
date: 2024-10-14 16:00:00 +0900
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

In this blog post I'll write about what I saw and felt during analysis, it's neither something covers entire VMProtect's feature nor in-depth, but more like initial brief research to get used to it. Cuz that'd be beyond my wheelhouse.

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

I noticed that the binary was built with `/DYNAMICBASE` option, which enables ASLR[^aslr]
It made me unpleasant in debugging session.

It's pretty easy to turned it off, open the binary with your favorite hex editor and navigate to 

```
NTHeader -> OptionalHeader -> DllCharacteristics -> IMAGE_DLLCHARACTERISTICS_DYNAMIC_BASE
```

and set it to 0.

Then go to Windows security in windows settings and in `App & browser control` tab, turn off `Randomize memory allocations (Bottom up ASLR)`.

That way you can turn off the ASLR and fix the image base across reboot.

> also setting `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\kernel\MitigationOptions`'s third significant bit to 0 does the same thing in registry.
{: .prompt-tip }

## [+] Confirm vm entry indication

Right off the bat, look at the main function.

![main](main.png)
_main function_

Right before going into vm_entry it's assigning value to the 3 registers.
It's not a deafult x64 __fastcall calling convension but come on it's VMProtect!
It'd do anything dirty.
you can assume that they could be used in vm_entry.
Keep it in mind and carry on.

```nasm
mov     r8d, 3
mov     edx, 2
mov     ecx, 1
```

Now let's look at the vm_entry (image below).
As you can see it's pushing 2 registers before going into a subroutine which I remember it was encrypted `vip` (like `rip` but in VM context) in VMProtect2, but not sure this time, `r14` is 0 and `r9` is holding a stack memory address that was initialized in `_initterm` function which doesnt look odd to me.
No idea if it's related to vmp or not.

![vm_entry](vm_entry.png)
_vm entry_

Btw in the nature of vm_entry, it should have a specific initialization process.
Since it's going to use general registers in its vm, it first push all the registers to stack to preserve the current state for later.
However, based on my memories of VMProtect2, I overconfidently thought 'This should be easy' and ended up learning a painful lesson.

In vmp3, the contents of vm_entry were finely chopped up, and it seemed that a single operation was performed through multiple functions.
In the VMP2 I had seen before, all register pushes were done in a single function, but this time even the pushes were divided into multiple functions and basic blocks, making it very difficult to search through

I managed to find it, as I said it was all separated, but the image below is the biggest piece of function which pushes many general registers.
The function has a branch in the middle of it however it turned out to be a part of the obfuscation, Opaque predicates[^opaque_predicates]. (Virtualized subroutines contain numerous amount of them lol)
I drew comments in the picture why it'd always takes same branch.

![vm_entry2](vm_entry2.png)
_red: opaque predicates, green: register push_

## [+] Locate VIP initialization

Many instructions in virtualized subroutines doesn't make sense to me and it confuses me at the same time. Idk what to look for and shit.
However 1 thing I know was vmp must've stored custom bytecode for vm somewhere in the binary.
And wherever it is, the virtualized process will eventually have to access it.

So I decided to debug the program and closely monitor the registers and stack to see if any of them started holding the bytecode pointer, usually refered to as `vip` (virtual instruction pointer).
It felt like took ages to find it, but I finally discovered it! In the image below, `r9` contains `0x100000000`, and `r10` holds the encrypted `vip`.
`r10` has been decrypted along the way, and by combining it with `r9`, the decryption process has been completed.

```
0xFFFFFFFF0004114D + 0x100000000 = 0x04114D
```

![vip](vip.png)
_virtual instruction pointer is being decrypted_

And following is where the ptr was pointing to.

![bytecode](bytecode.png)
_this is what vmprotect custom bytecodes look like_

To ensure it's correct one, I kept going and look for instructions that uses `r10`. Then I found the part where it retrieve a byte from `vip-7` and moving vip forward. 

![vip ref](vip_ref.png)
_getting a byte from vip and updating pointer_

vmp stores opcode and oprand in bytecode field. So when it wants to execute `add eax, 5`, retrieve each opcode and operand just like I showed you, and execute it.

> Note that vip looks technically going backward just like stack does, but apparently it varies depending on the vm you're dealing with. Some vms actually go forward and others go backward.
{: .prompt-tip }

## [+] Virtual stack initialization

Here it seems allocating new stack frame for vm.

![virtual stack](virtual_stack.png)

Moving `r12` to `rsp`, `r12` is pointing to stack memory, and the gap between `r12` and  `rsp` is 0x260.
By doing this it's creating exclusive stack of size 0x260 while preserving current stack at the same time.

![virtual stack dbg](virtual_stack_dbg.png)

## [+] Evil vm handlers

This is the one unit of vm handler.
Even tho it should be semantically one routine, it's chopped up to lots of basic blocks and it's using disgusting way to jmp around without executing whole subroutines it lands after each call/jmp.

Since it's too complicated, I couldn't just take entire picture of them, I manually put them together to one code below so let's investigate it closely!!

```nasm
0000000000044792        mov     eax, 21322628h
0000000000044797        neg     rax
000000000004479A        mov     r11, [rbx+rax*2+42644C50h]
00000000000447A2        lea     r8, [rsp+rax*2+42644D10h]
00000000000447AA        movzx   r9d, al
00000000000447AE        lea     rbp, [rax+rax+750119A6h]
00000000000447B6        mov     [rax+r8+21322628h], r11
00000000000447BE        lea     r8, ds:0FFFFFFFF9E3824A7h[rbp*2]
00000000000447C6        mov     rbp, [rbx+rax*2+42644C58h]
00000000000447CE        jge     loc_C2AF1
00000000000447D4        movsx   r11d, r8w
00000000000447D8        lea     rcx, [rsp+rax*2+42644C60h]
00000000000447E0        mov     [rcx+rax+21322628h], rbp
00000000000447E8        call    sub_8695C

000000000008695C        mov     r11, [rax+r10+21322620h]       ; r11 = (QWORD)(*vip - 8)
0000000000086964        movzx   ebp, ax
0000000000086967        xor     r11, rdi
000000000008696A        rol     r11, 1
000000000008696D        push    rax
000000000008696E        neg     [rsp+rax*2+8+arg_42644C43]
0000000000086976        mov     [rsp+r9*2+8+var_1B8], 0B9884282h
0000000000086982        not     r11
0000000000086985        lea     r8, ds:0FFFFFFFFEABD1E31h[r9*4]
000000000008698D        sal     ebp, 78h
0000000000086990        call    sub_83C34

0000000000083C34        bswap   r11
0000000000083C37        xadd    bp, ax
0000000000083C3B        rol     r11, 2
0000000000083C3F        neg     r11
0000000000083C42        mov     ecx, r8d
0000000000083C45        rol     rbp, cl
0000000000083C48        xor     rdi, r11
0000000000083C4B        mov     [rsp+rax*2+arg_42660008], 0CBD428Eh
0000000000083C57        mov     [rbx+rax*2+42660008h], r11
0000000000083C5F        movsx   r11d, al
0000000000083C63        dec     ebp
0000000000083C65        pop     r9
0000000000083C67        movzx   r8d, byte ptr [r10+rax+2132FFF7h]   ; r8d = (BYTE)(*vip - 9)
0000000000083C70        xor     [rsp+r11*8-8+arg_2], r11d
0000000000083C75        jnb     loc_CBAC0

00000000000CBAC0        xor     r8b, dil
00000000000CBAC3        inc     rbp
00000000000CBAC6        dec     r8b
00000000000CBAC9        call    sub_9DCE3

000000000009DCE3        xor     r11b, cl
000000000009DCE6        call    sub_84C20

0000000000084C20        ror     r8b, 1
0000000000084C23        mov     r9d, 3C8CAD36h
0000000000084C29        mov     edx, 3703F011h
0000000000084C2E        not     r8b
0000000000084C31        rol     r8b, 1
0000000000084C34        dec     bpl
0000000000084C37        setp    cl
0000000000084C3A        pop     rcx                       ; rcx gets return address on top of the stack
0000000000084C3B        add     rcx, 4ABEh                ; add constant offset to it
0000000000084C42        jmp     rcx                       ; jmp

00000000000A27A9        xor     dil, r8b
00000000000A27AC        lea     r8, [rsp+r8+arg_10]
00000000000A27B1        mov     rdx, [r8+rax+21330000h]
00000000000A27B9        pop     rax                       ; rax gets return address 0xCBACE
00000000000A27BA        add     rax, 0FFFFFFFFFFFBE97Eh   ; add constant offset to it
00000000000A27C0        jmp     rax                       ; jmp

000000000008A44C        lea     r8, [r11+r9-5DDEB1F0h]
000000000008A454        mov     [rbx+r11*2-122h], rdx
000000000008A45C        and     r8w, [rsp+r11-88h]
000000000008A465        call    sub_B0A97

00000000000B0A97        rol     [rsp+r11*2+var_116], 0ACh
00000000000B0AA1        mov     eax, [r11+r10-9Eh]             ; rax = (DWORD)(*vip - 13)
00000000000B0AA9        lea     rcx, ds:0FFFFFFFFFD9592B7h[r8*4]
00000000000B0AB1        ror     r8w, 24h
00000000000B0AB6        shr     r9w, 0C4h
00000000000B0ABB        lea     r10, [r10+r11*2-12Fh]          ; vip is no longer used in this vm handler, thus it moves pointer forward by 13
00000000000B0AC3        xor     eax, edi                       ; rax decryption starts by xoring with rolling key (edi)
00000000000B0AC5        movzx   edx, r11w
00000000000B0AC9        shr     r11, 44h
00000000000B0ACD        xor     r8b, [rsp+r11+arg_3]
00000000000B0AD2        inc     eax                            ; rax decryption
00000000000B0AD4        ror     eax, 1                         ; rax decryption
00000000000B0AD6        xor     eax, 8F8E1812h                 ; rax decryption
00000000000B0ADB        or      bp, r8w
00000000000B0ADF        not     eax                            ; rax decryption
00000000000B0AE1        mov     [rsp+r11+7], rdi
00000000000B0AE6        xadd    dl, r8b
00000000000B0AEA        push    r9
00000000000B0AEC        xor     [rsp+r11*2+6], eax
00000000000B0AF1        mov     rdi, [rsp+r11+0Fh]
00000000000B0AF6        mov     [rsp+r9+8+var_3C8C0AD3], rcx
00000000000B0AFE        xor     ebp, 0BDB8BB35h
00000000000B0B04        movsxd  rax, eax
00000000000B0B07        jge     loc_46A00

0000000000046A00        pop     r9
0000000000046A02        adc     rsi, rax                       ; add rax + carry flag to rsi which calculates final rsi
0000000000046A05        and     [rsp+rbp*2+var_1C112186], dl
0000000000046A0C        mov     r11, [rbx+rbp*8-70448650h]
0000000000046A14        mov     rax, [rbx+rbp*2-1C11218Ch]
0000000000046A1C        pop     r9
0000000000046A1E        adc     r11, rax
0000000000046A21        add     [rsp+r8-8+arg_2152D471], r9b
0000000000046A29        lea     rbp, ds:0FFFFFFFFDB094088h[r8*4]
0000000000046A31        mov     [rbx+r8+2152D477h], r11
0000000000046A39        lea     rbx, [rdx+rbx-1Dh]
0000000000046A3E        mov     [rsp+r8-8+arg_2152D477], 733392B5h
0000000000046A4A        neg     bp
0000000000046A4D        mov     [rsp+r8*2-8+arg_42A5A8DE], rsi ;rsi holds next vm handler address
0000000000046A55        retn    8
```

#### 1. Eye catching characteristics

As you can see it employs numbers of disturbing ways to obfuscate its component.

Firstly, at each memory access it does add/subtract operation with registers + constants to hide offset from static analysis.
This way you can't predict where exactly it's accessing.
Moreover making it difficult to search specific disassembly you're looking for by searching sequence of bytes.
Luckly despite of the big numbers it uses as constant, the result of calculation is usually simple.

```nasm
lea     rbp, [rax+rax+750119A6h]
mov     [rax+r8+21322628h], r11
lea     r8, ds:0FFFFFFFF9E3824A7h[rbp*2]
mov     rbp, [rbx+rax*2+42644C58h]
```
{: file='example.asm'}

Secondly, it exits current subroutine it's executing at every first `call` or `jmp` instruction it runs into.
Doesn't matter how big subroutine currently it's in, as soon as it encounters `call`/`jmp` it carries on to another location abandoning the rest of the subroutine and never came back to where it was.

Lastly, it frequently use indirect jmp using general registers such as `rax`, `rcx` or `r9`.
Among registers, a register is randomly picked and used for jmp.
However there's a unique tendency.
The jmp destination is always made out of return address pushed onto the top of the stack from last `call` + constant. 

```nasm
pop     rax                       ; rax gets return address 0xCBACE
add     rax, 0FFFFFFFFFFFBE97Eh   ; add constant offset to it
jmp     rax                       ; jmp

~~~~~~~~~~~~~~~~~~~~~~~~~~

pop     rcx                       ; rcx gets return address on top of the stack
add     rcx, 4ABEh                ; add constant offset to it
jmp     rcx                       ; jmp
```
{: file='example.asm'}

## [+] Deadstore removal plugin

By the way, the amount of deadstore vmp inserts between legit instructions are insane that I almost lose my temper so I quickly made an IDA plugin to remove all the deadstores in the current function. 

[vxcall/deadstore-remover](https://gist.github.com/vxcall/1b2841370d07f25dc0d729985306bf4f)

Place it in IDA's `plugins` folder.
It's utilizing a library called [triton](https://github.com/jonathansalwan/Triton), therefore you have to build it to use my plugin.
Also it works on IDA 8.X but doesnt work on IDA 9.X due to the API compatibility.
You can use it by placing cursor on the middle of a function in IDA and run the plugin I named it "Function Deadstore remover", so that the plugin will analyze and nop out every garbage instructions.

![deadstore-remover](deadstore_remover.png)
_nop out deadstore, if the instruction made out of multipul bytes, it truncates them_

## [+] Devirtualize it

Enough research, it's time to devirtualize it.
The word 'devirtualize' here doesn't mean recompiling it to clean binary.
But lift it to LLVM IR to see what virtualized subroutine does.
Recompile is another subject I hopefully will do one day!

For it I used [NaC-L/Mergen](https://github.com/NaC-L/Mergen) which is a very cool tool to disassemble the obfuscated binary and lift it into deobfuscated LLVM IR[^llvm_ir] by optimizing it.
That way Mergen can universary handle not only VMProtect but other obfuscators and virtualizers.

> I found it so cool that I made a couple of small pull requests to contribute it! You have to check it out too!
{: .prompt-info }

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

- `rcx`: 1
- `rdx`: 2
- `r8`: 3

respectively.

So the final equation will be `1 - 2 + 3 = 2`
vm_entry turned out to be returning 2.

## Conclusion

![footer](footer.jpg)

I discovered many interesting techniques and tendencies it has, and barely understand what it's doing in virtualized routines.
However advanced things patching and hooking virtualized function, or recompile it to devirtualized binary is yet another difficulties I'll have to figure out.
But thanks to this research I'm pretty sure I'll be capable of it one day.

After everything I've been through, I still feel VMProtect is over my head but gained priceless knowledges about it and I'm totally satisfied with it!

## References

[https://www.mitchellzakocs.com/blog/vmprotect3](https://www.mitchellzakocs.com/blog/vmprotect3)
[https://blog.back.engineering/17/05/2021/](https://blog.back.engineering/17/05/2021/)
[https://blog.es3n1n.eu/posts/obfuscator-pt-1/](https://blog.es3n1n.eu/posts/obfuscator-pt-1/)

## Footnotes

[^bin2bin]: **bin2bin**: a type of obfuscation software. stands for binary to binary. There's another type linker level obfuscation according to [es3n1n's Blog](https://blog.es3n1n.eu/posts/obfuscator-pt-1/).
[^aslr]: **ASLR**: Address Space Layout Randomization. When it is enabled, the binary will be loaded at random location each time which means you have to rebase the program in IDA everytime open it with x64dbg.
[^opaque_predicates]: **Opaque predicates**: a type of obfuscation techniques which generates a seemingly legit branches that is actually a garbage branching taking you the same branch every time. 
[^llvm_ir]: **LLVM IR**: LLVM's intermediate representation.

