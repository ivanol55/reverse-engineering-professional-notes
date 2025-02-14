# Debugging and disassembly techniques
In this section we'll go deeper into what exactly a **debugger** is and how we can use it, what a **disassembler** is, the difference between the two, and how to use different tools for those tasks like `Ollydbg` and `x64dbg`.

## Why do we need debuggers and disassemblers?
These tools provide a lot of **useful helpers** for our malware analysis tasks, such as:
- **breakpoints** and **process control**
- **step-in**, **step-out** and **step-over**
- **Lanching** and **attaching**
- debugging **32** and **64-bit** malware
- Debugging **DLLs**
- Comparing **decoded source** and binary

These are very useful tools that are very important on our malware reverse engineering and analysis journey. For example, the first instance of the worm Creeper, in 1971, was stopped on ARPANET because it was reverse engineered to understand how it worked, and a counterpart was developed to stop it, Reaper. Reverse engineering is the **leading technique** for **understanding the structure and operations of malicious programs** and what they're programmed to do. As always, malware becomes more sophisticated with time, and so do reverse engineering tools.

### Tool types and their purpose
Malware reverse engineering includes **disassembling** (and sometimes **decompiling**) malicious programs. During reverse engineering, binary instructions of the program are transformed into **code mnemonics** (higher-level constructs) so analysts can see what **actions** the program performs and what **system components** it affects. By knowing and understanding its operations, engineers can **create solutions** to mitigate its malicious effects.

We classify the different kinds of reverse engineering programs into distinct categories:
- **Disassemblers**
- **Decompilers**
- **Debuggers**

Debuggers and disassemblers are needed so we can **understand the functionality of malicious programs** when the source code is unavailable to us. They greatly **simplify** the task of code analysis and are used to determine the nature and purpose of a piece of malware, like classifying it as an information stealker, keylogger, rootkit, and so on. They help us understand **how the system was infected** and the **impact** it had on the environment, discover **host-based indicators of compromise**, including **filenames** and **registry keys**, which can be used as **signatures** for infection detection. We can also extract **network** indicators we can use on network monitoring suites to detect malware activity, like a C2 server IP as a malware activity trace.

PE viewers are another useful tool we have previously explored on these notes, used to view information on windows Portable Executable files, like their metadata or dependency viewing. Examples are `CFF Explorer`, `PE Explorer`, `PE Bear` and `pestudio`. Network analyzers are another type of tool that should be on our radar to capture networking traces and explore them later, the most widely used being `Wireshark`. While they are not a debugger, they can greatly help in our debugging tasks. The more information we can relate, the better.

### Process control, stepping
Disassemblers specifically **translate the machine code of the application into assembly**. These are used for static analysis, a technique discussed earlier on these notes, used for code interpretation that allows us to **understand the program without running it**. An example of a disassembler is IDA Pro. Decompilers, on the other hand, transform binaries into **high-level code** or **pseudocode**. This code is easier to read and understand. Debuggers, besides code disassembly, allow the reverser to **execute the target program in a controlled manner**, like executing specific instructions instead of the entire program. This allows us to see the program's execution flow and gain insights into its functionality. Widely used debuggers are, for example, `Ollydbg`, `Immunity Debugger`, `x64dbg`, `GDB` and `windbg`.

Another notable difference between debuggers and disassemblers is that a disassembler shows the program's state **before it is executed**. We are presented with the program's low-level assembly functions, or pseudocode if we use a decompiler. A debugger, on the other hand, shows **memory locations**, **registers** and **function arguments** *during the program's execution*. It further allows the reverse engineer to **change** them when the program is running to see how that change affects the malware sample. We can, for example, **modify a control flag**, **instruction pointer**, or the **code instructions** themselves. A good use is **skipping a function**, like avoiding a call to a particular function call with a breakpoint. Note that this may affect the program's stability.

### Launching and attaching
To start a debugger, we can either launch a new process or attach it to an existing process. To launch is to **run** a program using a debugger. The running program stops immediately before the execution of its main entrypoint after which control is paused and given to the reverse engineer. On the othert hand, to attach means to **bind** a debugger to an already running program. This pauses all the program's threads and passes control to the debugger. This is commonly used to debug a process that is infected by malware, but **we will miss the initial actions** that occur before the debugger is attached.

### Debugging DLLs
To debug a DLL, we need to find its **entrypoint**. This can be done by following these steps:
1. Use a debugger to **launch** or **attach** the process to a host process
2. When a DLL is loaded using `x64dbg`, the debugger will **drop an executable** on the same folder where this DLL is saved
3. This dropped executable will act as a host process to execute the DLL
4. When the DLL is loaded, we need to pause execution at the entrypoint of the DLL (we can configure this DLL breakpoint on the debugger)

This way we can get a sttarting point just when the DLL is about to run.

### Process control
Debuggers give the reverse engineer the ability to control or change the behavior of a process as it is running. Using debuggers, we can control the program execution and interrupt the program execution via breakpoints. We can **continue execution**, **step into** or **step over**, **execute until return**, or **run to cursor**. Let's look at these functionalities a bit deeper.

### Continue execution
Using this option, the debugger will execute program instructions until it reaches a breakpoint or an exception occurs. If we use this option without a breakpoint, the debugger will execute all instructions without granting any granular control. Hence this option is used with breakpoints, to pause when one is reached.

### Stepping on/over/into
Step on is a technique that runs every instruction step by step. This is **very slow** and may slow you down with **unimportant details**. For this, we use **step over**, which runs function calls as a single instruction block, decreasing the code amount by "stepping over" some functions. That said, **make sure this does not make you miss important functionality**, especially in cases where the stepped over function does not return anything. Check the CPU registers and values after a step-over!

**Step into**, like step over, executes a single command. However, step into enters a function call when encountered and execution is transferred to the first instruction inside the called function. This **helps us understand the inner working of a function** and not just its return values like step over does. This is also a useful technique when a function doesn't return anything.

### Execute until return
The debugger here will execute all instructions of the current function until it encounters a return instruction. You may used it in case you stepped into a function accidentally, or if the function turns out not to be interesting or useful and you want to get out of it and move on.

### Run to cursor
The debugger here will execute all instructions until it reaches the current location of the cursor or reaches the selected instruction. THis is similar to breakpoints, where the debugger allows us to interrupt code execution at a particular location of the code. A program that sttops at a breakpoint is called **broken**. An example is, if you can't figure out where a call instruction is redirecting the flow to, add a breakpoint on the call and check the contents of EAX, which will hold the target memory location.


### Breakpoints
Breakpoints can be set at a particular **instruction**, a **function**/**API call**, or when accessing **certain memory addresses** (**reading** from, **writing** to or **executing**). You can set several breakpoints and the execution will stop when any of them are triggered. There are several types of breakpoints:
- **software** breakpoints, where the debugger overwrites the first byte of the instruction with `INT 3` (opcode `0xCC`). Upon trigger, the control of the program passes to the debugger. You can set unlimited software breakpoints, but advanced malware searches for this instruction and **changes behavior** if it finds it, making analysis difficult
- **hardware** breakpoints, specific **CPU registers** that allow us to interrupt the execution of the program using debug registers. The control is transferred to the debugger when memory is accesses for reading or writing. The CPU debug registers are `DR0`-`DR7` (`0`-`3` contain the breakpoint line addresses, `4`-`5` are reserved, `6` holds the debug status and `7` determines the activation mode which can be **read**, **write** or **execute**)
- **memory** breakpoints, used to interrupt execution if memory is accessed for read or write, useful to find out when a meory address is accessed and by which instruction
- **conditional** breakpoints, which break **in case a condition is satisfied**. It can cause slowness in the execution, and it is offered by the debugger, not by the CPU, so you can determine a condition for software or hadrware breakpoints.

## Debugging 32-bit and 64-bit malware
The main difference between these two is that 64-bit malware uses **extended registers**, **64-bit memory addresses and pointers**, and have **slightly different conventions for API calling**. For 32-bit functions, when the arguments are pushed, the stack **grows** and if they are popped, the stack **shrinks**. For 64-bit functions, the space of the stack is **allocated at the beginning of the function** and it's not modified until the end of the function. This space is used to store function parameters and local variables.

## Source code vs binary debugging
We perform analysis on malicious binaries since the source code for binaries is generally not available. Source-level debugging is performed on the **high-level languages**, usually on IDEs. Both allow looking at program instructions and setting breakpoints to examine values and memory locations. Source code analysis depends on the programming language, and it is **much easier** than analysis of compiled binaries. Binary debugging requires **reverse-engineering** and **low-level language** programming skills.

## Debugging and debuggers: crash course
We'll explore a very widely used tool here, `Ollydbg`. `Ollydbg` is a **free**, **GUI-based** debugger, that only contains user-level debugging mode (f we want kernel-level debugging, the only tool currently available that supports it is `Windbg`). It can extend its functionalities using **plugins**. It is generally run with **administrative privileges** to see what malware could do with full privileges on the system.

When an executable is loaded in, we will see a main window displaying **CPU registers**, **virtual memory addresses** and **memory contents**, **related opcodes**, **assembler mnemonics**, and information the debugger can identify, such as **interesting function calls**. We can open additional tabs with more information, such as what **executable modules** has the sample loaded. we'll see the **entry point address**, the **module name**, **file version**, and the **file path** for each loaded module. All this information is for the main thread mainly, but we can see what other threads the program is running.

We can also see what system resource **handles** are loaded in and being used and what **windows** has the sample spawned.

As mentioned earlier, we can use this software to set breakpoints on the individual instructions of the program as necessary. We can also, for example, when we're stopped at a breakpoint, **follow the EAX register to the memory dump** to see what the memory it's about to use looks like and what's in it.

There are similar tools like `Immunity Debugger` with similar features. There is also other software suites that include more advanced debuggers, like `IDA Pro`, which includes different debuggers for different types of applications and architectures. These are more complete, but **a lot more complex** compared to `Ollydbg` or `Immunity`. Another great alternative, open-source with advanced features, is `x64dbg`.

If you want a multi-platform, open license reversing IDE, we recommend `Ghidra`. `Radare2` is also a good option, although more complex.