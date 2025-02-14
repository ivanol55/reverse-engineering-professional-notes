# The necessary theory: CPU Registers
## Introduction
This first section aims to cover all of the **necessary theory** as well as the concepts on which the practical side of the course is based. We will start by setting some definitions, why you might need to reverse engineer something, and then we'll proceed into the technical details. In closer detail, in this section we'll discuss:
- Intel IA-32 CPU archictecture (**x86**)
- The **stack**
- The **heaps**
- **Exceptions**
- Windows **APIs**
- Windows **internals**
- some common types of reversing **tools**

Keep in mind some x86 assembly instructions will be referenced here, but we will not go into much detail about them as this is not an assembly programming course. **Mastering Assembly to become a reverse engineer is not necessary**. Of course knowing low-level languages like C or C++ can be helpful, but in regards of reversing assembly language, you just need to understand what you see, so you can reproduce it in another language you're more comfortable with.

## What is reverse engineering?
Reverse engineering is the **understanding of the internals** of something made by a human, through analysis, **without having access to its design principles** and the way its components interact with each other in order to make it work. In other words, it's the process of **taking apart something that someone else built and understanding how they did it**, partially or completely, so that you are able to make something on your own that can achieve the same purpose. In this course we focus on software, but this also applies to hardware.

## Do we need reverse engineering?
Imagine you have some old software that is now unsupported and you need it, and it has a **bug you could fix**, but you don't have access to the source code. Or a that you have a passion for **malware analysis**. You won't be able to access the malware code, but you still want to know how it works, how it infects your computer and avoids the antivirus. There are several cases where, either for work or as a hobby, reverse engineering comes in very handy.

## The basics behind the intel IA-32 CPU Architecture
A CPU has a lot of components that run it on its architecture. We are mostly interested in **registers**, which are **small units of internal memory** that are stored inside the CPU. These registers being embedded inside of the CPU gives them the perk of being **really fast to access** to retrieve the data inside them. 

This makes them really useful to **perform basic mathematic instructions very fast**, like **addition**, **substraction**, **multiplication** and **division**, as well as performing **logic gate operations** such as OR, AND, XOR, NOT, etc. However it should be noted that **their size capacity is very limited** and in this architecture we're focusing on, there's just **eight registers that we can use for generic purposes**, so called general purpose registers, **each 4 bytes (32 bits)** in size. 

### General purpose registers
Let's have a deeper look into the registers that we are most interested on while reverse engineering applications.
- **EAX**: Accumulator for operands and results data
- **EBX**: Pointer to data in the DS storage segment
- **ECX**: Counter for string and loop operations
- **EDX**: I/O pointer
- **ESI**: Pointer to data in the segment pointed to in the DS segment register; source pointer for string operations
- **EDI**: Pointer to data (or destination) in the segmen pointed to by the ES register; destination pointer for string operations
- **EBP**: Pointer to data on the stack (in the SS segment)
- **ESP**: Stack pointer in the SS segment register

We can further narrow down access in some pointers:
- **{A,B,C,D}L** accesses the first 8 bits of the register (0 to 7)
- **{A,B,C,D}H** accesses the second 8 bits of the register (8 to 15)
- **{A,B,C,D}X** accesses the first 8 bits of the register (8 to 15)
- **{S,B}P** accesses the first 16 bits of ESP and EBP respectively
- **{S,D}I** accesses the first 16 bits of ESI and EDI respectively
We cannot individually access the first 16 bits of each register (what's called the high byte and low byte). Here's a more full reference:

```
  31            16       8       0     First 16-bit block     Full 32-bit register
 |---------------|--AH---|---AL---|            AX                    EAX
 |---------------|--BH---|---BL---|            BX                    EBX
 |---------------|--CH---|---CL---|            CX                    ECX
 |---------------|--DH---|---DL---|            DX                    EDX
 |---------------|-------BP-------|                                  EBP
 |---------------|-------SI-------|                                  ESI
 |---------------|-------DI-------|                                  EDI
 |---------------|-------SP-------|                                  ESP
```

### the EFLAGS register
Another very important CPU register that we need to mention is the **EFLAGS** register. This is a **collection of 1-bit values** used for **status**, **control** and **system** flags. Below we have a schema of this register:
```
                                31             16       8       0
                                |--------------------------------|
                                |0000000000IVVAVR0NIIODITSZ0A0P1C|
                                |----------DIICMF-TOOFFFFFF-F-F-F|
                                |-----------PF-----PP------------|
                                |------------------LL------------|
                                |--------------------------------|
                                           ^^^^^^ ^ ^^^^^^^ ^ ^ ^
ID flag (ID)-------------------------------|||||| | ||||||| | | |
Virtual Interrupt Pending (VIP)-------------||||| | ||||||| | | |
Virtual Interrupt Flag (VIF)-----------------|||| | ||||||| | | |
Alignment Check (AC)--------------------------||| | ||||||| | | |
Virtual-8086 mode (VM)-------------------------|| | ||||||| | | |
Resume Flag (RF)--------------------------------| | ||||||| | | |
Nested Task (NT)----------------------------------| ||||||| | | |
I/O Privilege Level (IOPL)--------------------------||||||| | | |
Overflow Flag (OF)-----------------------------------|||||| | | |
Direction Flag (DF)-----------------------------------||||| | | |
Interrupt Enable Flag (IF)-----------------------------|||| | | |
Trap Flag (TF)------------------------------------------||| | | |
Sign Flag (SF)-------------------------------------------|| | | |
Zero Flag (ZF)--------------------------------------------| | | |
Auxiliary Carry Flag (AF)-----------------------------------| | |
Parity Flag (PF)----------------------------------------------| |
Carry Flag (CF)-------------------------------------------------|
```
These flags are used in different processes:
- Positions with 0 or 1 on them should not be touched as they are **reserved**
- **Arithmetic instructions** make use of the **OF**, **SF**, **ZF**, **AF**, **PF** and **ZF** flags. On the other hand, the `SCAS` (scan string), `CMPS` (compare string - `cmsb`, `cmpsw`, `cmpsd`) and `LOOP` (`LOOPE`, `LOOPZ`, `LOOPNE`, `LOOPNZ`) instructions make use of the **ZF** flag in order to indicate the completion of their operations (and in some cases the result), For example, the `repe cmpsb` instruction compares two strings, byte by byte, and if the two strings are equal, the **ZF** flag is set to `1`. Otherwise, it will be set to `0`.
- The **Control** flag **DF** is used to control instructions related to string processing. If **DF** is set to `1`, the string instructions atuo-decrement, so that they are **processed from higher to lower addresses**. The reverse happens if this control flag is set to `0`, going from lower to higher memory addresses. We usually set it to `0` with the `cld` instruction, or set it to `1` with the `std` instruction. It's important that **ESI** and **EDI** register point to the start or the end of the strings **before starting this operation**, and **ECX** must contain the number of bytes we wish to compare.
- The **system flags** and the **IOPL** field inside the **EFLAGS** register are involved with operating system operations. **The most important one is the Trap flag**, which enables single-step mode. This generates a single-step exception after the execution of each instruction, and is **critical for debugging** while keeping control of stepped execution.

### Segment registers
In addition to the common registers and flags, we also have a group of **16-bit registers** called segment registers which **contain special pointers** called **segment selectors** that identify the different types of segments in memory. In order to access a particular segment in memory, the appropiate segment register must contain the correct segment selector.

The following schema demonstrates the segment registers of the **flat memory model** used by Windows NT OS, which is structured so that applications see available physical memory as an array of memory locations. **The OS takes care of the rest of controls**, such as not letting applications access kernel memory or the addresses of other applications so they don't interfere.

```
                   Segment 
                  registers
 |----------------|  CS --------|                 |-----------------------------|
 |----------------|  DS --------|                 |                             |
 |----------------|  SS --------|                 | Overlapping segments of     |
 |----------------|  ES --------|                 | up to 4 Gigabytes of memory |
 |----------------|  FS --------|                 | beginning on address 0      |
 |----------------|  GS --------|                 |                             |
                                |---------------->|-----------------------------|
```

Each one of the segment registers points to a specific type of storage: **code**, **data** or the **stack**. More information on the stack in a later section. As shown in the figure above, the **CS** register contains the segment selector for the **code segment**, which is the **memory area that stores the instructions** that are being executed.

**DS**, **ES**, **FS** and **GS** registers point to four different data segments, which are used to store different types of data strctures or single variables. Finally, the **SS** register points to the **stack segment**, where the stack of the currently executing thread is stored in memory. For this reason, **all stack-related operations use the SS register** to locate the stack segment.

### The instruction pointer register
We also have the **Instruction pointer register (EIP)**, also called the **Program Counter (PC)**, pointing to the *next* instruction to be executed in the code segment. Every time an instruction is executed, the **EIP** is updated to point at the next instruction. This register cannot be read directly so if we need to check its value, we'll need to use a trick with stack-reading we'll explain later.

### Debug registers
Debug registers, as their name implies, are **used to control the debug operation** of the processor. There are eight of them labeled from **DR0** to **DR7**.

```
 31             16       8       0
 |--------------------------------|
 |LLRRLLRRLLRRLLRR00G001GLGLGLGLGL|  
 |EEWWEEWWEEWWEEWW--D---EE33221100|    DR7
 |NN33NN22NN11NN00----------------| register
 |33--22--11--00------------------|
 |--------------------------------|

 31             16       8       0
 |--------------------------------|
 |1111111111111111BBB011111111BBBB|  
 |----------------TSD---------3210|    DR6
 |--------------------------------| register
 |--------------------------------|
 |--------------------------------|

 31             16       8       0
 |--------------------------------|
 |--------------------------------|  
 |--------------------------------|    DR5
 |--------------------------------| register
 |--------------------------------|
 |--------------------------------|

 31             16       8       0
 |--------------------------------|
 |--------------------------------|  
 |--------------------------------|    DR4
 |--------------------------------| register
 |--------------------------------|
 |--------------------------------|

 31             16       8       0
 |--------------------------------|
 |--------------------------------|  
 |----------Breakpoint 3----------|    DR3
 |---------Linear address---------| register
 |--------------------------------|
 |--------------------------------|

 31             16       8       0
 |--------------------------------|
 |--------------------------------|  
 |----------Breakpoint 2----------|    DR2
 |---------Linear address---------| register
 |--------------------------------|
 |--------------------------------|

 31             16       8       0
 |--------------------------------|
 |--------------------------------|  
 |----------Breakpoint 1----------|    DR1
 |---------Linear address---------| register
 |--------------------------------|
 |--------------------------------|

 31             16       8       0
 |--------------------------------|
 |--------------------------------|  
 |----------Breakpoint 0----------|    DR0
 |---------Linear address---------| register
 |--------------------------------|
 |--------------------------------|
```

In the context of referse engineering, **we are mostly interested in the first four** debug registers (DR0 to DR3), which are used to **store hardware breakpoints** on specific addresses which will be triggered if a desired condition is met. In other words, if a specific type of memory access occurs, the execution of the program will pause, giving us the opportunity to examine it under the debugger.

For example, we can set a **hardware breakpoint** on memory access inside the address space of the process we're examining. We'll set this breakpoint to **trigger if this memory is read from or written to** by a CPU instruction.

We can also **set a hadrware breakpoint on execution** on a specified memory address where the executable code is placed, which will be **triggered on every attempt to execute instructions** from that memory allocation. This will allow us to see **step-by-step execution** of that code. Keep in mind **each thread has its own CPU context**, and a hardware breakpoint set for the memory area of that thread will not be triggered by other threads in the program. Memory area ***AND*** thread context have to match to trigger a hardware breakpoint.

Software breakpoints in the other hand work by substituting the original byte located in that address, where we set the breakpoint, with a 0xCC byte (INT 3h). Since this implies **modifying the code in memory**, these are completely independent from thread context, thus always effective regardless of thread count.

Keep in mind that debug registers are **privileged resources**, which means we cannot directly access them from Ring 3 (also called **Userland**) where software is normally executed. In order to set these hardware breakpoints in Windows, we use a **ring 3 API** which will transfer the execution to **kernel level** in order to update the debug registers.

### Machine-specific registers
These types of register are also called **Model-specific registers**. They handle system-related functions and they are not accessible to applications, except from the time-stamp counter. This is a 64-bit register, and its content can be read using the `RDTSC` instruction which stands for **Read time-stamp counter**. When read, the low 32 bits are loaded into **EAX**, and the higher 32 into **EDX**. This value is **increased every CPU clock cycle** and it resets to 0 when the processor is reset.
