---
title: How Game Cheaters Install Their Illegal Kernel Driver?
date: 2024-06-26 14:06:00 +0900
categories: [low level, kernel]
tags: [cpp, kernel]     # TAG names should always be lowercase
media_subpath: /assets/img/posts/how_game_cheaters_load_their_illegal_kernel_driver/
image:
  path: header.png
  lqip: header.svg
  alt: header
---

## Introduction

In recent years, AAA game titles have increasingly adopted sophisticated anti-cheat systems such as Easy Anti-Cheat, BattlEye, and Riot Vanguard.
These systems have become remarkably robust, primarily due to their utilization of kernel-level drivers.
Operating at this privileged level allows them to monitor the entire computer system more effectively than user-mode programs.

Despite the advanced nature of these anti-cheat systems, skilled game hackers continue to find ways to circumvent them.
Their approach often involves loading their own kernel drivers to suppress or bypass the anti-cheat mechanisms at the kernel level.
This ongoing cat-and-mouse game explains why cheating remains a persistent issue in online gaming.

This post will introduce you to a tool called [kdmapper](https://github.com/TheCruZ/kdmapper), which is frequently used by cheat developers to load unsigned kernel drivers into kernel space. Understanding this tool and its implications is crucial for comprehending the current state of game security and cheating techniques.

## Table of Contents

- [What is kernel driver?](#what-is-kernel-driver)
- [[+] Dark arts: kdmapper](#-dark-arts-kdmapper)
- [[+] Process of mapping](#-process-of-mapping)
  - [loads iqvw64e.sys](#loads-iqvw64esys)
  - [removes it’s trace for additional stealthiness](#removes-its-trace-for-additional-stealthiness)
  - [reads raw data of your kernel driver into memory](#reads-raw-data-of-your-kernel-driver-into-memory)
  - [maps your driver into kernel space](#maps-your-driver-into-kernel-space)
  - [manually calls your DriverEntry](#manually-calls-your-driverentry)
- [[+] kdmapper in action](#-kdmapper-in-action)
- [[-] It's detectable](#--its-detectable)
- [Conclusion](#conclusion)

## What is kernel driver?

Kernel drivers are specialized programs that operate differently from typical user applications.
While standard programs run in user mode, kernel drivers execute in the privileged kernel space.
They serve as a bridge between user-mode applications and hardware, responding to requests from user-mode programs to interact with system resources.
To draw an analogy, you can think of kernel drivers as similar to backend servers in web development, processing requests from user-mode "clients."

For a comprehensive understanding of kernel drivers, I highly recommend reading [Windows Kernel Programming](https://leanpub.com/windowskernelprogrammingsecondedition) by Pavel Yosifovich. This book offers in-depth insights into the Windows kernel and driver development.

![windows-rings](windows-rings.webp){: w="600" .center}
_windows ring system_

Windows offers user mode API for sending request to kernel drivers for example [DeviceIoControl](https://learn.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-deviceiocontrol), [WriteFile](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefile) and [ReadFile](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile), and then kernel drivers can run corresponding tasks that you define.

Even if you can develop a kernel driver, you cannot load it immediately - Windows lays down strong security measures and only allows drivers that have passed Microsoft's and other vendors' review and are officially signed to be loaded.

__So u can't load your driver? No, you still can. This signing system has a significant flaw. Let me explain later.

## [+] Dark arts: kdmapper

kdmapper is a user mode application to load your unsigned kernel driver onto your computer using volunerable signed kernel driver called `iqvw64e.sys` which is an old version of network diagnosis driver developed by Intel.

`iqvw64e.sys` used to be volunerable. It has strong assets in it including capability of calling kernel APIs to load a random kernel driver without **any access controls**. The volunerability is fixed already but the exploitable old version of driver is still recognized as 'signed' by Windows and that is a flaw I was talking about. The certification has expiry date of course, but to keep backward compatibility Windows decided to allow us loading expired signed driver too and ended up with being exploited even these days.

I wouldn't say kdmapper is undetected cuz it has been open sourced quite a while, so it's gonna be detected if you use it as is. I'll explain about it later.

But now, let's take a close look at how kdmapper load your driver to kernel memory!

## [+] Process of mapping

Its mapping process can be broken down into those steps:

- loads `iqvw64e.sys`
- removes it's trace for additional stealthiness
- reads raw data of your kernel driver into memory
- maps your driver into kernel space
- manually calls your DriverEntry

#### Loads iqvw64e.sys

First, it loads `iqvw64e.sys` inside [service::RegisterAndStart](https://github.com/TheCruZ/kdmapper/blob/30f3282a2c0e867ab24180fccfc15cc9b819ebea/kdmapper/service.cpp#L3) function. It sets up corresponding registries first and then uses native NT API NtLoadDriver like this.

```cpp
// Need to enable SE_LOAD_DRIVER_PRIVILEGE privilege
auto RtlAdjustPrivilege = (nt::RtlAdjustPrivilege)GetProcAddress(ntdll, "RtlAdjustPrivilege");
auto NtLoadDriver = (nt::NtLoadDriver)GetProcAddress(ntdll, "NtLoadDriver");

ULONG SE_LOAD_DRIVER_PRIVILEGE = 10UL;
BOOLEAN SeLoadDriverWasEnabled;
NTSTATUS Status = RtlAdjustPrivilege(SE_LOAD_DRIVER_PRIVILEGE, TRUE, FALSE, &SeLoadDriverWasEnabled);
if (!NT_SUCCESS(Status)) {
    Log("Fatal error: failed to acquire SE_LOAD_DRIVER_PRIVILEGE. Make sure you are running as administrator." << std::endl);
    return false;
}

std::wstring wdriver_reg_path = L"\\Registry\\Machine\\System\\CurrentControlSet\\Services\\" + driver_name;
UNICODE_STRING serviceStr;
RtlInitUnicodeString(&serviceStr, wdriver_reg_path.c_str());

// Calling NtLoadDriver with registory path
Status = NtLoadDriver(&serviceStr);
```
{: file='kdmapper/service.cpp'}

#### Removes it's trace for additional stealthiness

After that it tries to remove some traces that anti-cheat is checking in [intel_driver::Load()](https://github.com/TheCruZ/kdmapper/blob/30f3282a2c0e867ab24180fccfc15cc9b819ebea/kdmapper/intel_driver.cpp#L32).

Each of the function does following

ClearPiDDBCacheTable
: clearing driver name from list of drivers in `ntoskrnl.exe`. driver name'll be added when you load one

ClearKernelHashBucketList
: deleting driver name and hash of driver certificate from particular list in `ci.dll`

ClearMmUnloadedDrivers
: deleting driver name to prevent kernel from remember and add to unloaded driver list

ClearWdFilterDriverList
: unlinking driver name from a linked list in `WdFilter.sys` which holds all running drivers

```cpp
if (!intel_driver::ClearPiDDBCacheTable(result)) {
    Log(L"[-] Failed to ClearPiDDBCacheTable" << std::endl);
    intel_driver::Unload(result);
    return INVALID_HANDLE_VALUE;
}

if (!intel_driver::ClearKernelHashBucketList(result)) {
    Log(L"[-] Failed to ClearKernelHashBucketList" << std::endl);
    intel_driver::Unload(result);
    return INVALID_HANDLE_VALUE;
}

if (!intel_driver::ClearMmUnloadedDrivers(result)) {
    Log(L"[!] Failed to ClearMmUnloadedDrivers" << std::endl);
    intel_driver::Unload(result);
    return INVALID_HANDLE_VALUE;
}

if (!intel_driver::ClearWdFilterDriverList(result)) {
    Log("[!] Failed to ClearWdFilterDriverList" << std::endl);
    intel_driver::Unload(result);
    return INVALID_HANDLE_VALUE;
}
```
{: file='kdmapper/intel_driver.cpp'}


#### Reads raw data of your kernel driver into memory

Then it read your binary data into memory to calculate image size and header size from its nt header.

```cpp
std::vector<uint8_t> raw_image = { 0 };
if (!utils::ReadFileToMemory(driver_path, &raw_image)) {
    Log(L"[-] Failed to read image to memory" << std::endl);
    intel_driver::Unload(iqvw64e_device_handle);
    PauseIfParentIsExplorer();
    return -1;
}

// ...

const PIMAGE_NT_HEADERS64 nt_headers = portable_executable::GetNtHeaders(data);

// ...

uint32_t image_size = nt_headers->OptionalHeader.SizeOfImage;

// ...

DWORD TotalVirtualHeaderSize = (IMAGE_FIRST_SECTION(nt_headers))->VirtualAddress;
image_size = image_size - (destroyHeader ? TotalVirtualHeaderSize : 0);
```
{: file='kdmapper/main.cpp'}

#### Maps your driver into kernel space

[kdmapper::MapDriver](https://github.com/TheCruZ/kdmapper/blob/30f3282a2c0e867ab24180fccfc15cc9b819ebea/kdmapper/kdmapper.cpp#L73) function is responsible of actual driver mapping.

Before map, it allocates kernel memory as well as physical memory based on 3 options. All of them does allocation anyways in `AllocMdlMemory`, `AllocIndependentPages` or `intel_driver::AllocatePool`. Each of the method has their own advantages so u better research them.

After that it fix relocations just similar to what you do when u inject your dll in manual map way.

```cpp
// Write fixed image to kernel

if (!intel_driver::WriteMemory(iqvw64e_device_handle, realBase, (PVOID)((uintptr_t)local_image_base + (destroyHeader ? TotalVirtualHeaderSize : 0)), image_size)) {
    Log(L"[-] Failed to write local image to remote image" << std::endl);
    kernel_image_base = realBase;
    break;
}

```
{: file='kdmapper/kdmapper.cpp'}


#### Manually calls your DriverEntry

Finally it calls your custom DriverEntry.

```cpp
NTSTATUS status = 0;
if (!intel_driver::CallKernelFunction(iqvw64e_device_handle, &status, address_of_entry_point, (PassAllocationAddressAsFirstParam ? realBase : param1), param2)) {
    Log(L"[-] Failed to call driver entry" << std::endl);
    kernel_image_base = realBase;
    break;
}
```
{: file='kdmapper/kdmapper.cpp'}

The method it employs to call custom driver entry in [intel_driver::CallKernelFunction](https://github.com/TheCruZ/kdmapper/blob/30f3282a2c0e867ab24180fccfc15cc9b819ebea/kdmapper/include/intel_driver.hpp#L154) is very common technique in kernel exploit development but still interesting, so let's closer look at it.


Look at the source code below where it constructs shellcode called `kernel_injected_jmp`.

```cpp
uint8_t kernel_injected_jmp[] = { 0x48, 0xb8, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0xff, 0xe0 };
// Copying original shell code into original_kernel_function to restore NtAddAtom later
uint8_t original_kernel_function[sizeof(kernel_injected_jmp)];
// Replacing 0x00s with DriverEntry's address
*(uint64_t*)&kernel_injected_jmp[2] = kernel_function_address;
```
{: file='kdmapper/include/intel_driver.hpp'}

You might be wondering what is `{ 0x48, 0xb8, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0xff, 0xe0 }`, here's one by one description.

- `0x48` and `0xb8` represents x64 constant load into `rax` register
- The bunch of `0x00`s are filled with your DriverEntry's address in the last line.
- `0xff` and `0xe0` indicates `jmp rax`

So the shellcode's basically assigning your DriverEntry address into `rax`, then jumping to `rax`.
You can imagine it like this:

```nasm
mov rax, 0xFFFFF80179A3B000 ; suppose this is DriverEntry's address
jmp rax
```

After shellcode construction, it gets `NtAddAtom` from kernel and replace first 12 bytes with the aforementioned shellcode.
By replacing it, when a user calls `NtAddAtom` this kernel mode `NtAddAtom` will eventually be called and the hook will be kicked and it transfers further execution to your DriverEntry.

```cpp
// Getting kernel NtAddAtom's address
static uint64_t kernel_NtAddAtom = GetKernelModuleExport(device_handle, intel_driver::ntoskrnlAddr, "NtAddAtom");

// ...

// Overwrite the pointer with kernel_function_address
if (!WriteToReadOnlyMemory(device_handle, kernel_NtAddAtom, &kernel_injected_jmp, sizeof(kernel_injected_jmp)))
    return false;
```
{: file='kdmapper/include/intel_driver.hpp'}

> The hook target can be anything but `NtAddAtom` theoretically. However, if the target function has `_security_cookie` implemented then that's not a case and you have to avoid such functions if I'm not mistaken. Also it's safe to hook popular functioins, it's not gonna be spam called, because kdmapper will later restoring the original bytes immediately after calling a hook.
{: .prompt-tip }

Now all it has to do is calling `NtAddAtom` from user mode.
When syscall happens and the instruction transitions into kernel mode `NtAddAtom`, the kernel hook that we set will be kicked automatically.

For easy read I strip out some details but this is where kdmapper calls user mode `NtAddAtom`:

```cpp
// Getting uesr mode NtAddAtom from ntdll
const auto NtAddAtom = reinterpret_cast<void*>(GetProcAddress(ntdll, "NtAddAtom"));
// Making it callable
const auto Function = reinterpret_cast<FunctionFn>(NtAddAtom);

// Calling
*out_result = Function(arguments...);
```
{: file='kdmapper/include/intel_driver.hpp'}

I omit some details but these are the main steps it takes to map your driver.

## [+] kdmapper in action

I'm going to demonstrate how to map your kernel driver using kdmapper here.

Right off the bat, get or build kdmapper.exe in whatever way.

Next, build your driver. Suppose you have desired kernel driver source loaded in Visual Studio.
Go to project settings of kernel driver and configure the entry point.

- Configuration Properties
  - Linker
    - All Options
      - ✅ Entry Point -> DriverEntry

It initially should be GsDriverEntry. To let kdmapper call your custom driver entry point, **u need to rename it to DriverEntry**.

![custom_entry_point](custom_entry_point.png)
_Entry Point setting_

Once you build it with the custom entry point setting, you can make kdmapper do its magic by drag and drop the .sys file onto kdmapper binary. (unless you want to use options)

> Driver signing enforcement doesn't have to be disabled in this way but make sure no anti cheats or anti virus is running in your vm.
{: .prompt-tip }

![dnd_kdmapper](dnd_kdmapper.png)

There you go! your driver will be mapped in your kernel!

## [-] It's detectable

Well...unless you configure it well.

The thing is anti cheats has suspicious drivers list and periodically check if known volunerable drivers have been loaded and `iqvw64e.sys` is one of them. Moreover windows has list of similar concept, you can disable by editing `VulnerableDriverBlocklistEnable` registry key.
You have to exploit your own driver which is capable of read and write memory or some alternative APIs like MmMapIoSpace/MmUnmapIoSpace or ZwMapViewOfSection/ZwUnmapViewOfSection. This is called Bring Your Own Volunerable Driver, in short BYOVD lol.

In case you are interested in, [loldrivers.io](https://www.loldrivers.io/) is a website you can find volunerable drivers at for example. Of course it's the best to have your own driver not disclosed publically tho.

But most important and difficult part than BYOVD is **make your driver pretends like it's a legit driver**. If your driver behave bad, it will lead u banned. For example if you use normal communication method between user mode application and driver, it will be detected.

You have to adopt tricky method for your communication such as via:

- Sockets
- Shared memory
- Named pipes
- Data ptr

Besides that there're ingenius articles about [abusing large page](https://vollragm.github.io/posts/abusing-large-page-drivers/) and [bound hook](https://www.cyberark.com/resources/threat-research-blog/boundhook-exception-based-kernel-controlled-usermode-hooking) which also could be used.

It's all require of practice and try-error process
Hence, you have to learn, reverse engineer anti cheat and know what it's monitoring.

## Conclusion

![footer](footer.jpg)

I believe kdmapper is very powerful and still widely used method among game hackers.
However, to utilize it and detour anti cheat's check, u have to strive to make your driver look legetimate process.

Good luck:)