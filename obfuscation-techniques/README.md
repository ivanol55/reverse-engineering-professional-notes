# Obfuscation techniques
Apart from harming the target, malware will always want to **protect itself** so it doesn't get deleted, reverse engineered or stopped. For this, malware developers add additional layers of protection to defend against these investigation attempts. Many forms of defense exist, but malware mostly uses **obfuscation**. Obfuscation is a method that is used to **transform binary and text data into an unreadable form**, hard to understand by someone without context. This transformation makes the analysis or reverse engineering of the program difficult.

The obfuscation implementation can be simple, such as small **bit manipulations**, or advanced, such as using **cryptography**. Generally adversaries apply encoding and encryption to hide data. In some cases, obfuscation goes a step further in which the whole file is obfuscated using a special program called a **packer**. This is used by malware authors to hide critical words in the code stored as strings, since they might **give away important, traceable information** about the malware's behavior. Examples of such strings are usually registry keys or malicious URLs.

Malware authors need to know as many obfuscation techniques as possible to hide their malicious activity and to transfer the malware to the target easily without being detected. On the other hand, analysts need to know them so they can **decrypt or decode these obfuscated strings** and get to the real data to understand the malware's behavior.

While dissectting a malware, you may be interested in **identifying the encryption and decryption process**. You may need to determine the encryption function used and its associated keys. For example, if you need to identify the encryption technique of network contents, you may find the encryption function before the network output function `HttpSendRequest`. If you want to identify how the C2 downloaded content is decrypted, you may find the decryption function just before the retrieval through `InternetReadFile`.

Generally enconding and encryption are used for the following reasons:
- Hide **command and control communications**
- **Avoid signature-based detection** like IDS's or IPS's
- Conceal the **contents of configuration files** used by the malware
- Avoid detection by **encrypting exfiltrated information** from victim systems
- **Obscuring strings** on the malicious binary

## Decoding
Sometimes malware authors just use very **simple** algorithms to obscure their code. The advantages of these are:
- They are **easy to implement**
- They **don't require a lot of resources**
- They provide **reasonable protection** against security products and security analysts

Here's some simple encryption and enconding algorithms used. Keep in mind, these may be combined, mixed and matched to make analysis harder!

### Simple arithmetic
In these techniques some basic mathematical operations are used to encrypt and decrypt the file. For example, addition may be used for encryption and substraction for decryption, or vice versa. For example, if We have the encrypted data `0xF9 0x11 0x22` and the encryption key was adding `0x11`, the decrypted data is obtained by substracting that to each byte, resulting in `0xE8 0x00 0x11`

### Caesar cipher
This is one of the **earliest known** and **simplest** ciphers, also known as the **shift cipher**. The main concept of this method is to **substitute** each letter in plaintext with another by shifting it a fixed number of positions. A to Z represent 0 to 25, and if our key is 2, we move each letter 2 positions in the round: `infected` becomes `kphgevgf`. A quick way to calculate caesar cipher is to use modulus 26, which will give us the target position. If `i` is `8` and our key is `2`, we do `(8 + 2) % 26` and we get our new position, `10` which is `k`. To reverse it, we reverse the operation: `(10 - 2) % 26`, which gets us back to `8`, so `i`. Some malware authors use a bigger character set, like APT1 did when creating `WEBC2-GREENCAT`, that includes lower-case, upper-case, numbers and symbols and uses a key of `56`.

### the XOR operation
This one is the **most used and easiest obfuscation technique**, as it is very simple to implement for hiding data from untrained eyes. Take the example string `lppt>++sss*ahwiet*gki+bmhaw+wpeca6*a|a`. The data is not readable in that for, but applying `XOR` with the value `0x4` we can see the string `https://www.eslmap.com/files/stage2.exe`, a potential binary download. `XOR` is a bitwise operation and is applied on a corresponding set of bits of the operands. Using XOR operation, if both bits are the same, the result is `0`, otherwise it's `1`. For example, the XOR of `3` and `5` is:

```
 ------------
 | 3 | 0011 |
 | 5 | 0101 |
 |---|------|
 | 6 | 0110 |
 ------------
```

#### Single-byte XOR encoding
A variant is the single byte `XOR`, where we apply this rule on each byte with the encryption key. The word `security` with the key `0x30` will run the operatiton with each lettter byte to result in `CUSEBYDI`. An interesting property of this method is that it's reversible. If you have the key, using it on `CUSEBYDI` with key `0x30` will return `security` again.

Breaking this method is **simple and trivial** even when the key is unknown, as we can iterate through every single-byte key very quickly with tools like **XORSearch** or **CyberChef**.

#### Multi-byte XOR encoding
Using single-byte XOR gives us a key space of 255, which makes bruteforcing it trivial. Therefore, adversaries tend to implement **multi-byte XOR**, immune against this brute-forcing technique. If a 4-byte key is used for data encryption, then we will have a key space of `4,294,967,295` keys (until `0xFFFFFFFF`).

#### XOR deobfuscation
To avoid detection by the tools mentioned before, malware authors implement different **tricks** to make detection harder. A simple one is to use a **two-cycle approach** that makes a second `XOR` pass using a different key. Another one is **using a loop** to increase the `XOR` value.

### Base64 encoding
This technique allows encoding binary data into an ascii string format. TThe standard base64 encoding set consists of 26 uppercase letters, 26 lowercase letters, 10 numbers and + and / for new lines (`ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/`). Every 3 bytes or 24 bits of the binary data is encoded into four characters from this set. Each encoded character size is 6 bits.

To encode a value in base64, we convert the string into values based on the table mentioned earlier, numbering 1 to 63, so `infected` becomes `105 110 102 101 99 116 101 100`, we convert that to binary `01101001 01101110 01100110 01100101 01100011 01110100 01100101 01100100`, we regroup them as 6-bit blocks `011010 010110 111001 100110 011001 010110 001101 110100 011001 010110 010000` (added zero-padding at the end), then convert that back to ascii with the table: `aW5mZWN0ZWQ=` (the equal is padding).

Sometimes attackers may use modified variants of base64, like removing padding `=` characters, breaking the decoding if you don't pay attention. Other change `+` and `/` to `-` and `_`. Others flip or reorder the base64 encoding table.

## Unpacking packed malware
Malware authors use **packers** and **cryptors** to obfuscate their binaries by transforming the resulting file into a form that is hard to analyze. This provides a better shield to **protect their samples** from anti-virus detection and complicates the reverse engineering and static analysis of the code. A **packer** is a piece of software that obfuscates an executable by **compressing** its contents and generating a new executable altogether, while a cryptor **encrypts** the executable instead of compressing it.

Without unpacking, a lot **less information** than usual will be available that make basic static analysis less useful:
- **less strings** will be revealed by default by PE analyzers
- We will see a lot **fewer imported functions**
- The **program instructions** will be obscured

To make this analysis possible you will need to **deobfuscate** and **unpack** the sample. We can use tools to detect packers such as `PEiD` or `DiE`. If we dig further into the program structure, as we explored in earlier entries of these notes, we will find references to packed code and a small section called the **unpacking stub** that is in charge of unpacking the malware upon execution. As a malware analyist it is important that we can understand and reverse engineer unpacking stubs.

An unpacking stub has three steps:
1. **Unpack** the packed executable into memory
2. **Find the import table** for the original executable
3. **Change the execution of the program** to the original entry point so the unpacked malware starts running, generally jumping with an obfuscated function or using OS functions sich as `NtContinue` or `ZwContinue`

There are four main approaches that are generally used to unpack an executable file:
### `LoadLibrary` and `GetProcAddress`
This is the most common method, which works by importing the `LoadLibrary` and `GetProcAddress` functions. After the unpacking, the stub can read the import information of the original file and load each necessary library using those two functions.

### Not resolving imports
In this second approach, the packer does not tamper with the original import table, so the unpacking stub does not resolve the imports. In this approach the **loader will load the DLL and imported functions** in memory and their addresses. The compression in this method is not optimal and lacks stealth because the imported table is stored in plaintext.

### One function per library
In this approach, one import function is kept in the original import table for each DLL. This way, only **one fuction from each imported library is revealed**. Though stealthier than the previous approach, imported libraries are revealed during analysis. In comparison with the first approach, it is simpler, since the unpacking stub doesn't do library loading, but it still needs to resolve most of the functions.

### Removing all imports
This is the stealthiest approach, where all imports are removed, including `LoadLibrary` and `GetProcAddress`. The packer needs to search for all functions needed and rebuild the import table, and use them to unpack the file. Or it can search for `LoadLibrary` and `GetProcAddress` and use them to find all other libraries. This search is dfone using function hashes to avoid detection.

## How to unpack the sample
To unpack a packed malware, you can either use an **automated tool**, or **manually unpack it**. Although automated unpacking saves time, it is **not completely reliable**. While manual unpacking is **time-consuming**, it is more reliable and an important skill to have on your abilities.

There are three options for unpacking:
1. **Automated static** unpacking (preferred. It is the fastest and restores the original executable without running it)
2. **Automated dynamic** unpacking (The next best thing, but it has to run the sample)
3. **Manual dynamic** unpacking (figure out the packing algorithm yourself, reverse it, fump the process into the disk and fix up the PE header manually)

There are some open and very extended packers, commercial and open source, like `UPX`. `ASPack`, `Petite` and `PECompact`. However, malware developers tend to either modify these packers to avoid easy unpacking or build it themselves. This usually leaves more sophisticated malware with manual unpacking work.

## Anti-analysis techniques
Several different techniques are used to defend against the malware analysts understanding the sample. Here's several examples:

### Anti-debugging techniques
Anti-debugging and anti-analysis techniques are widely used by malware to avoid malware analysts decoding how the malware works and stopping it. Some techniques used to identify if a debugger is running are:
- **The Windows API**, which provides some APIs to detect debuggers, like `IsDebuggerPresent` or `CHeckRemoteDebuggerPresent` (we can bypass this by setting the `BeingDebugged` value to `0` with DLL injection before executing the ckecking code)
- the Kernel32 `CloseHandle()` or `NtClose()`function, which will raise an exception when an invalid handle is passed. This allows the malware, due to the function's implementation, to detect an attached debugger. We can bypass this with a **FirstHandler Vectored exception** that will handle the exception and resume execution
- the `FindWindow()` function, used to find debugger windows running in the system of popular debuggers like `WinDbg`
- the `NtGlobalFlag` flag of the PEB structure, used to check if it's under a debugger (value `0x70`) or not (value `0x00`). We can avoid detection by setting the flag to `0x00`.
- **Timing defense**, where the program will check the time being taken to execute instructions. If it is found to take longer than usual, it can be an indication of being debugged.
- **Software breakpoint detection**, where the malware will search for INT 3 (`0xCC`) bytes, indicating a software breakpoint from a debugger. This is usually checked by a function checksum, which we can find and replace with a constant
- **Hardware breakpoint detection**, where the malware will generate an exception it can handle, which sends it all the register, including the debug register, to check if a debugger is running. We can avoid this detection by setting the debug registers to `0`.
- **Self-debugging**, where a process will create another process to try to debug itself, which will fail if it already has a debugger attached.
- **Debugging messages**: staring from Windows 10, the OutputDebugString function calls RaiseException with particular parameters. As a result, the output exceptions of the debugging should be handled by the debugger. Two exceptions can be used to check for a debugger: `DBG_PRINTEXCEPTION_W(0X4001000A)` and `DBG_PRINTEXCEPTION_C(0X40010006)`. If the exception is handled, a debugger is attached.
- **Structured Exception Handling** is a technique to handle both software and hardware exceptions. If the program is being debugged, the debugger will intercept the control after the `INT 3h` interruption, if not, the SEH handler will receive the control.
- **TLS Callbacks**: most debuggers will start the entry point of the program according to the PE header. The Thread Local Storage callback approach is used to run code instructions before or after the execution of the main application code. This code, generally anti-debugging code, will run before reaching the entry point and being noticed by the analyst
- the **isDebugged** field found in the second byte of the Process Environment Block. It is set by the system when the process is debugged. This byte can be reset to 0 without consequences for the course of the execution, as it is informative.
- **Interrupts** are conditions that pause the processor temporarily to process another task. Malware may detect some common false interrupts for breakpoints, such as `INT 3`, `INT 0x2C` as a debug assertion exception, `INT 0x2D` which raises `EXCEPTION_BREAKPOINT` if no debugger is attached, or `ICEBP` (`0xF1`) which generates a single-step exception.
- The **trap flag**, residing in the `EFLAGS` register, which raises a single-step exception `INT 0x01h` after executing an instruction. If the flag is set to 1, the CPU will produce this flag after each instruction execution. The trap flag is ignored in debug mode, which will make the program enter the wrong execution flow and exit

### Anti-disassembly techniques
Anti-disassembly techniques are used by malware authors to prevent or delay the process of reverse-engineering their malware. Some of these techniques are:
- **API Obfuscation**, which changes the identifier names of function names in the code, such as class or method names, into random names. This makes the code harder to understand.
- **Opcode/Assembly code obfuscation** means to encrypt sections of the code and instructions, making them harder to read.
- **Junk/spaghetti code** is a technique used to complicate the function flow, which makes it harder to read
- **Control flow graph flattening** aims to flatten the control flow of functions by breaking up the nested loops and if-statements. then it hides each of them in a large switch statement case that is wrapped inside of a loop body.
- **Jump instruction with the same target** means that an unconditional jump is used to jump into its own address then continue, which the disassembler doesn't understand isnce it disassembles one instruction at a time.

### General obfuscation
There are general obfuscation techniques widely used through the code, such as using **decimal numbers** throughout the assembly code instructions. Sometimes the binary form of an instruction is used instead of the assembly instruction, like changing `MOV DL, 75` to `db 10110010 b`. This way, generally harmful instructions detection is made harder for the analyst.

### Anti-VM obfuscation
This branch of techniques consists of attempting to detect if the malware is being executed in a **virtual machine**, which usually indicates an analyst is trying to understand the sample in a **volatile environment**. Studying these techniques will help **harden the analysis VM environment** against these evasion techniques. We should reduce the **VM indicators** as much as possible to make it harder for malware to identify and bypass the VM.

Virtual machines have several artifacts that make it clear we are running a sample inside of a virtualized environment. Such artifacts may be:
- **files**, like certain drivers and DLLs for mouse control or VM tray icons installed by the hypervisor
- **registry keys**, as hypervisors create certain system keys for their drivers or device enumeration
- **processes** that the hypervisors run on the guest, like `Vmwareuser.exe` or `vboxservice.exe`
- **services** that load the drivers mentioned earlier, or tools like virtual disk helpers
- **network device adapters**, like the known mac address id's of certain virtualization hypervisors
- Checking **CPU instructions** (CPUID, hypervisor brands, MMX)
  - if `CPUID` is run with input `EAX=1`, the return value determines the feature of the processor. If the first **31** bits of `ECX` are `0`, it's a physical machine, if they're `1`, it's a VM.
  - If it's called with `EAX=0` and it's a physical machine, it will return **AuthenticAMD** or **GenuineIntel** depending on the CPU, however VMs return the hypervisor name, like **Hyper-V** or **VMWareVMWare**
  - if `CPUID` is run with input `EAX=40000000`, it will get the virtualization vendor string saved in `EAX`, `ECX`, `EDX`. For example, Microsoft gets `Microsoft HV`
  - `MMX` is the set of instructions used for faster processing of graphical applications. Usually these instructions are not supported inside of a VM, so their absence might indicate virtualization
- A VMWare **magic number**, where VMWare communication with the host is done through a particular I/O port, so malware can test that, and if that communication is successful, it means it's a virtualized environment

We counter these with **anti-anti-VM techniques**, where we can prevent these detections by **manipulating CPUID instruction results** of the target VM from the host machine. Most vendors allow the host to modify the CPUID and CPU features. This is possible because each time the VM executes a `CPUID` instruction a VM-Exit occurs and the hypervisor passes the execution to VMM. Hence, this allows us to man-in-the-middle modify these results and avoid detection.

## Process hollowing
**Process hollowing**, also known as **process replacement** or **Runpe**, is a process in which the malware unmaps or hollows out the code of a legitimate process from memory and overwrites the memory space of the victim process with its malicious code. The contents of the `PEB` are the same, but the actual data and code of the process are changed. The path of the process being hollowed remains the same.

For example, when performing process hollowing of `svchost.exe`, the path will remain `C:\Windows\system32\svchost.exe`, but the executable section in memory will be replaced by the malware code. Using this technique an attacker can avoid detection by host IPS solutions and firewalls. The process is as follows:
1. **Create a legitimate process** in a suspended state that the malware can hollow
2. **Unmap the in-memory code** sections of the legitimate process
3. The malware will **allocate new memory on the target process** to host the malware code and writes the malware's sections into the target process memory space
4. The malware will then **set the entrypoint of the hollowed process to the malware's entrypoint**
5. The malware finally **resumes the hollowed process**, which runs the payload loaded in memory
