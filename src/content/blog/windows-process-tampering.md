---
title: "Windows Memory Tampering 101: An Introduction to Process Manipulation"
author: Asier Núñez
pubDatetime: 2024-12-29T04:06:31Z
slug: windows-process-tampering
featured: true
draft: false
tags:
  - Windows
  - Memory Tampering
  - Security
  - Ethical Hacking
description: "An introduction to memory tampering techniques, their applications, and the practical lab setup for learning the basics of process manipulation in Windows."
---

## Table of contents

## Motivation

A year ago, while at university, I was invited to host a lecture on process tampering. I had an hour and a half to explain and demonstrate basic memory tampering techniques. After discussing with the professor who offered me this opportunity, we decided that a practical lab would be more effective for this kind of topic. I created a lab where attendees could follow along with hands-on exercises.

The lecture turned out to be a great success — or so I was told! 😊

This post aims to revisit that lecture and teach you the basics of process tampering.

## Lab Setup

To follow along and try things yourself, you’ll need to set up your environment with the following tools:

- Windows 10/11
- [Windows SDK](https://learn.microsoft.com/en-us/windows/win32/desktop-programming#get-set-up)
- [Visual Studio](https://visualstudio.microsoft.com/) with the `Desktop development with C++` package.
- [Cheat Engine](https://www.cheatengine.org)
- [Visual Studio](https://code.visualstudio.com/) or your choice code editor.
- If using VSCode, I recommend this extension pack: [C/C++ Extension Pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools-extension-pack)

For the lab, we will use a simple target program — a GUI application with a counter and three buttons: Increment, Decrement, and Clear. You’ll compile this program yourself. To do this, install Visual Studio (the purple one, not Visual Studio Code) with the Desktop development with C++ package.

Here’s how your setup should look:

![setup](@assets/images/windows-process-tampering/wpt-visual-studio-setup.png)

> **Note:** _[This](https://learn.microsoft.com/en-us/windows/win32/desktop-programming#get-set-up) is the official Microsoft guide for Windows App development with the Win32 API_

We will debug process memory with [Cheat Engine](https://www.cheatengine.org/) (other tools such as [x64dbg](https://x64dbg.com/) should work if you prefer it).

> **Waring:** _Be aware that Cheat Engine installer contains AdWare_

As for code editor, we wil be using Visual Studio Code in this post but any code editor will work.

You’ll also need to clone the repository with its submodules:

```shell
git clone https://github.com/Asiern/procmem-manipulation --recurse-submodules
```

## Process Tampering Basics

Before diving into code, let’s cover the fundamental concepts of memory tampering and how processes are managed in Windows.

### What is memory tampering?

Memory tampering refers to techniques that modify the memory of a running process to alter its behavior. These techniques can be used for various purposes, such as:

- Modifying memory values: For example, changing an internal variable in real-time to bypass a validation check.
- Altering instructions: For instance, skipping a security check by modifying the code’s execution flow.
- Injecting code: This involves adding custom functionality to a process by injecting a DLL or other code into its memory.

### How Does a Memory Tampering Attack Work?

When you open a program, the operating system loads it into RAM. Windows uses a technique called [Virtual Memory](https://en.wikipedia.org/wiki/Virtual_memory), which gives each process its own isolated address space. Key points include:

- Each process gets a separate virtual address space.
- Virtual memory maps to physical memory (RAM) via a page table managed by the OS.
- Memory protection mechanisms prevent one process from accessing another’s memory.

Memory tampering bypasses these protections to modify the target process. A typical tampering attack involves these steps:

1. Locating the target process.
2. Connecting to the process.
3. Locating the memory space to modify.
4. Performing the memory tampering.

#### 1. Locating the target process

Each Windows process has a unique PID (Process Identifier). For this lab, we will be searching for a window title to get the PID.

> _Note:_ If the process doesn’t have a visible window, other techniques may be required.

#### 2. Connecting to that process

Once we have the PID (Process ID), we need to obtain the base address of the module we want to tamper with. You may be wondering: What is a module?

When you open a program, it is loaded into memory. Often, these programs are composed of several parts (or modules) that are loaded into memory, each with a base address.

So in this part we will need to find the target module's base address inside the virtual memory.

#### 3. Locating the memory space to modify.

This step often requires reverse engineering knowledge and tools like Cheat Engine or x64dbg. You'll need to analyze the process’s memory to locate the desired addresses.

#### 4. Performing the memory tampering.

Once we located the target addresses, we just need to call the read or write functions from our code pointing to those addresses.

## Process Tampering Lab

### Target Process

Let's start by compiling and running our target process. After intalling Visual Studio and the Windows SDK as mentioned before, we can compile the target process binary. This can be achieved by multiple ways, in my case, I will be using the CMake VSCode Extension to configure and compile the project. Once intsalled the CMake extension pack, if we run the `CMake: Configure` command (Ctrl + Shift + P to open the command menu) we will be asked to select a kit as shown in the following picture.

![CMake kit setup](@assets/images/windows-process-tampering/wpt-visual-studio-code-kit.png)

In my case, as I'm running a 64 bit system I will select the amd64 kit. Once selected, CMake will start generating the project files inside the build folder.
Once the project files are generated, we need to select which target we want to build, in the case of this project, there are two targets: playground and demo. Playground is the target process so we will select it.

![building-playground-target](@assets/images/windows-process-tampering/wpt-playground-build.png)

This should have genrated the binary `build/playground/Debug/Playground.exe`. If we execute it, we will see the following:

![playground-gui](@assets/images/windows-process-tampering/wpt-playground.png)

### Reverse Engineering

Now that our target is up and running, we can connect with Cheat Engine to start looking for our target virtual memory addresses.
We can connect Cheat Engine to a procress as shown in the following image.

![Connecting-with-CE](@assets/images/windows-process-tampering/wpt-cheat-engine-hook.png)

Once connected, we can start looking for values.

Before getting into how to use Cheat Engine, we should take a look at our target process to make thinks simpler.
As you could have seen by playing with the GUI, the playgorund APP lets you increase, decrease and clear a counter value.
The following code fragment it what makes that possible.

```cpp
// Show main window
{
    static float f = 0.0f;
    static int counter = 0;

    ImGui::Begin("Playground");

    ImGui::Text("counter = %d", counter);
    if (ImGui::Button("Increment"))
        counter++;
    ImGui::SameLine();
    if (ImGui::Button("Decrement"))
        counter--;
    if (ImGui::Button("Clear"))
        counter = 0;

    ImGui::End();
}
```

As we can see in the code, there is a variable called counter defined as static. We need to make a pause to explain the different kind of offsets that we can find.
An offset is the distance between the base address and our target, if our base address was `0x1` and we had an offset of `0xA`, our target address would be `0xB`.
From now on, we will be working with offsets and adding them to the module base address we mentioned earlier.

Continuing from where we left off, it's important to understand the difference between static and dynamic memory addresses. When a program is compiled, the compiler assigns memory addresses for the variables based on how they are defined in the code. Depending on how a variable is declared, its memory address can either be static or dynamic, and this has significant implications when discussing memory manipulation.

#### Static vs Dynamic Memory Addresses

In languages like C and C++, variables that are defined with a keyword such as static or those that are defined globally have a static memory address. This means the memory address of these variables is determined at compile-time and remains constant throughout the execution of the program.

For example, in the code from the previous example, the variable counter is defined as static:

```cpp
static int counter = 0;
```

This means the memory address of counter is assigned at compile-time and remains fixed during the execution of the program. The memory addresses of static variables are easy to find and manipulate because they are known and constant at all times.

On the other hand, in languages like C#, the memory addresses of variables are assigned dynamically at runtime. Instead of being fixed at compile-time, the memory addresses of these variables can change over time, depending on factors like memory usage and garbage collection (GC).

For example, in C#, variables of value types like int or float might be assigned dynamic memory addresses, particularly when they are allocated on the heap (e.g., when using new keyword to instantiate objects). These memory addresses are harder to track because they can change as the program runs, and the memory management is handled by the runtime (garbage collection).

We need to keep this in mind when approaching a tampering attack as this will define how we will search for memory values. For this lab, as code is written in C++, memory addresses will be static and we will not need to think about it.

> **Note**: When facing a program with dynamic addresses, one of the best approaches we can use to search for pieces of code (variables or instructions) is to perform AOB (Array Of Bytes) Scans. The idea, in essence, is to search for an AOB that is unique so regardless on where is located we will be able to find it.

### Reading Memory

Now that we have a basic understanding of memory addresses, let's start by reading the counter value from the target process.

First, we need to find the base address of the module we want to tamper with. In this case, the module is the playground.exe process. We can use the `GetModuleBaseAddress` function we defined earlier to get the base address of the module.

```cpp
DWORD pid = ProcUtils::GetProcId(L"Playground");
uintptr_t moduleBase = ProcUtils::GetModuleBaseAddress(pid, L"Playground.exe");
```

> **Note**: To list all the modules in a process, we can use the `EnumProcessModules` function or Cheat Engine (Memory View > Tools > Dissect PE headers).

Once we have the base address, we need to find where the counter variable is stored. We can use Cheat Engine to find the memory address of the counter variable. Cheat Engine lets us search for values in memory and filter them based on their type (e.g., 4-byte integer, float, etc.). In this case, we know the counter is an integer, so we can search for a 4-byte integer value. We also know it's value, so we can search for an exact value.

![](@assets/images/windows-process-tampering/wpt-ce-scan0.png)

As we can see in the previous image, Cheat Engine found 2936 possible addresses that match the value 0. We can now increment the counter and search for the new value.

![](@assets/images/windows-process-tampering/wpt-ce-scan1.png)

After incrementing the counter, we can see that the number of possible addresses has decreased to 1. This is the address we are looking for. We can check if this is the correct address by changing the value in Cheat Engine. If the counter in the GUI changes, we have found the correct address. To work with addresses is recommended to add them to the address list as shown in the following image.

![](@assets/images/windows-process-tampering/wpt-ce-add-to-list.png)

> **Note**: Cheat Engine displays static memory addresses with a green color.

If we take a look at the previous images, we can see that the search results displays the value as `Playground.exe+0x16C3C0`. This is the offset we were talking about earlier.

> **Note**: The offset is the distance between the base address and the target address. This value could be different in your case as it depends on the compiler.

```cpp
uintptr_t counterAddr = moduleBase + 0x16C3C0;

// Initialize Memory Class
mem = new MemUtils(pid);

// Read memory
int counterValue = mem->readMem<int>(addr);
std::cout << "Counter Value: " << std::dec << counterValue << std::endl;
```

> **Note**: All off the code snippets in this post are part of the `main.cpp` file in the `demo/src` folder of the repository.

### Writing Memory

Writing memory is as simple as reading it. We just need to call the `writeMem` function from our `MemUtils` class.

```cpp
// Write Memory
int newValue = 100;
mem->writeMem<int>(addr, newValue);
```

### Disable Assembly Code

Another common memory tampering technique is to disable assembly code. This technique involves changing the assembly code of a process to alter its behavior. For example, we can disable the increment button in the playground app by changing the assembly code that handles the button click event. We can do this by patching the assembly code with a NOP instruction. A NOP instruction is an assembly instruction that does nothing when executed.

> **Note**: More on NOP instruction [here](<https://en.wikipedia.org/wiki/NOP_(code)>).

To disable the increment button, we need to find the assembly code that handles the button click event. We can use Cheat Engine to find the assembly code and its memory address. This can be done by right-clicking on the address we added to the address list and selecting "Find out what writes to this address" as shown in the following image.

![](@assets/images/windows-process-tampering/wpt-ce-what-writes.png)

After selecting the option, we will be asked if we want to attach the debugger to the process. We can select "Yes" and a new window will open as shown in the following image.

![](@assets/images/windows-process-tampering/wpt-ce-what-writes-menu.png)

When an instruction that writes to the address is executed, the debugger will pause the process and show the assembly code that writes to the address. Now we can increment or decrement the counter and see the assembly code that writes to the address.

![](@assets/images/windows-process-tampering/wpt-ce-what-writes-instruction.png)

In this case, as we can see in the previous image, the instruction that writes to the address is:

```asm
mov [7FF7EAF4C3C0],eax
```

If we press on `Show disassembler` we will see the following:

![](@assets/images/windows-process-tampering/wpt-ce-disassembler.png)

We can see that the instruction dedicated to write the counter when the increment button is pressed is located at offset `0x42E`. Now lets look at the previous instructions to see if we can find something interesting.

```asm
Playground.main+424 - 74 0E          - je Playground.main+434
Playground.main+426 - 8B 05 045E1600 - mov eax,[Playground.exe+16C3C0]
Playground.main+42C - FF C0          - inc eax
Playground.main+42E - 89 05 FC5D1600 - mov [Playground.exe+16C3C0],eax
```

From the previous code we can see that the instruction that writes to the counter is located at offset `0x42E`. But we can also see that there is an `inc` instruction at offset `0x42C`. This instruction is the one that increments the counter. We can disable the increment button by patching the `inc` instruction with a `nop` instruction.
To do so, we will use the `patch` function from our `MemUtils` class.

```cpp
// Instruction: inc eax (FF C0)
uintptr_t incAddr = baseAddr + 0x42C;

// Disable increment button
mem->patch(incAddr, (BYTE *)"\x90\x90", 2); // replace inc eax with nop nop
```

This code will replace the `inc eax` instruction with two `nop` instructions like this:

```asm
Playground.main+42C - 90 90          - nop nop
```

> **Note**: If we wanted to re-enable the increment button, we would need to patch the `nop` instructions back to `inc eax`. Or we could just restart the process.

### Code Caves

Code caves are another memory tampering technique that involves injecting custom code into a process. A code cave is an area of memory that is unused or contains code that is not executed. We can use code caves to inject custom code into a process without modifying the original code. This technique is useful for adding new functionality to a process or bypassing security checks.

To inject code into a process using a code cave, we need to find a suitable location in memory where we can inject our code. We can use Cheat Engine to find code caves in a process or allocate memory.

Once we find a code cave, we need to get the address of the code cave and write our custom code to that address. After writing the code, we need to redirect the flow of execution to the code cave. This can be achieved by several methods, such as changing the return address of a function or using a jump instruction.

In this case, we will change the increment button's instruction for a jump instruction that will redirect the flow of execution to our custom code.

Going back to the inrement instructions, we need to decide if we are mantaining the original functionality or if we are going to replace it. In this case, we will
mainain the original functionality and add a new one. Before starting to write instruction, we need to know how far we are going to jump and the space we have to write our jump instruction, in x86 architecture, there are different types of jumps: short, near and long.

> **Note**: More on jump instructions [here](<https://en.wikipedia.org/wiki/JMP_(x86_instruction)>).

In the following image we can see the schema of a code cave. Instructions 1, 2 and 3 are overwriten with the jump instruction and relocated to the code cave where we can write our custom code and then jump back to the original instructions once we are done. This way we can add new functionality to the process without modifying the original code.

![code-cvave](@assets/images/windows-process-tampering/wpt-code-cave-schema.png)

As how to inject the code, you should now have the knowledge to do so with Cheat Engine. If you want to do it programmatically, it requires a bit more of work but it is possible, although I recommend you first try to achieve it with Cheat Engine. If you have doubts or get stuck, these are the steps you should follow (every step can be done with Cheat Engine):

1. Allocate memory in the target process.
2. Write down the address of the memory allocated (as this will be your code cave).
3. Write down the address of the instruction you want to replace.
4. Write down the instructions you will overwrite.
5. Overwrite the instructions with a jump instruction that redirects the code cave.
6. Write your custom code in the code cave.
7. Restore the original instructions (those that were overwritten) in the code cave.
8. Redirect the flow of execution back to the original instructions, with another jump instruction.

## Beyond the Basics

This post covers the basics of memory tampering and provides a practical lab setup for learning the basics of process manipulation in Windows. Memory tampering is a powerful technique that can be used for various purposes, such as modifying memory values, changing the flow of execution, and injecting code into a process.

If we want to go further, we can explore more advanced memory tampering techniques, such as: DLL injection, hooking, kernel-level tampering, and more. These techniques require a deeper understanding of Windows internals and reverse engineering, but they can be very powerful when used correctly.

I hope this post has been helpful in introducing you to the world of memory tampering. Happy hacking! 🚀
