# Static analysis techniques: the common procedures
In this section we'll start learning about dynamic analysis techniques when looking into malware samples. We will be focusing on the behavior of malware and why we need to dynamically analyze it, the different dynamic analysis methods, the pros and cons of each one, and tools we can use to make this task easier. We'll look at different kinds of malware and their behaviors (Downloaders, droppers, keyloggers...) and go over the methids they use to operate their peyloads, such as injection techniques or LoLBins

But first, let's discuss what exactly is dynamic analysis, the different types of dynamic analysis, when to use each of them and their differences. We'll then dive into the different types of resources used in this task.

## What is dynamic analysis?
Until this point in the notes, we understood we could analyze malware using static methods, wether looking at their signatures, what functions or system APIs they used, or disassembling them and looking at low-level code. But if those do not reveal enough information to answer our questions, we need to go deeper.

For example, if the sample is applying some obfuscation, getting packed with a custom packing mechanism we don't have access to, or extending its capabilities by downloading modules from outside, we'll need to run the sample to gather data about what it actually does at runtime. This is where dynamic analysis comes into play.

Dynamic malware analysis is analyzing malware samples by running them in a contained environment. We will run the malware and see what it does, how it behaves, what exactly it is harming, what operations are being executed and how those are being performed. We will observe the services and processes that are being affected during execution, if the malware is hiding and how.

Another reason it's called dynamic analysis is that there are so many possibilities that the execution can bring, that it is up to the analyst to track, monitor and understand what is happening as the sample runs. As long as we are prepared and understand what we're doing, we'll be able to find some answers.

### Where to run the samples
We can run the samples in several different kinds of isolated environments:
- Virtual Machines, simulated computers inside of a host computer that can run a pre-configured state, saved in a point in time, and can be rolled back as necessary. This ease of use and predictability makes it a preferred option today
- Separate physical systems allow us better simulation of real environments to avoid anti-analysis VM detection, but is a lot more expensive to reproduce. In this case we usually use tools to be able to restore the machine to an earlier state when rebooted, like Shadow Defender, DeepFreeze, Sandboxie or Rollback Rx Professional
- Automated sandbox software allows us to run the samples inside of a snadbox in case we can't have a dedicated environment, which also has the perk of running automated reports on the potential findings it gets from the execution. An example here is Cuckoo, which we will cover later.

### Feeding the malware
An important part of dynamic analysis is adapting to the changes required by the malware. To get the proper information out of it, we may need to change our environment so the malware thinks it has a potential target and runs to its full potential. Make it think it's on its target and not in an isolated environment for analysis.

We can define the states of the contained environment in two categories:
- Insecure environment, meaning we lack many security measures on purpose, to see how the malware acts against an easy target. This also means running the sample as privileged to see its capabilities when running as a high integrity process.
- Hardened environment, where we configure a secure machine we would see in a properly configured environment and see if our real environment would survive an attack from the sample you have. By doing so you may find hidden exploitable misconfigurations or vulnerabilities we could patch.

### When to use each technique
Before we start executing malware, we should have a good understanding of the pros and cons of both static and dynamic malware analysis. Let's cover a couple of points we should keep in mind, but remember it is a good idea to be able to do both, as required to have more tools under your arsenal.

Here's a reference table for these techniques:
```
          --------------------------------------------------------------------------------------------------------
          |              Method          |   Difficulty |                       Notes                            | 
----------|------------------------------|--------------|--------------------------------------------------------|
|         | Header analysis and hashing  | Easy         | Fast / inexpensive                                     |
|         | PE Analysis                  | Intermediate | Fast / inexpensive                                     |
| Static  | Using scanners               | Easy         | Fast / inexpensive                                     |
|         | Reverse Engineering          | Difficult    | Slow / expensive / understanding of assembly           |
|         | Static code analysis         | Difficult    | Slow / expensive / understanding of assembly           |
|---------|---------------------------------------------|--------------------------------------------------------|
|         | Running the malware sample   | Intermediate | Relatively fast / Not very expensive                   |
| Dynamic | Running the sample sandboxed | Easy         | Relatively fast / Not very expensive                   |
|         | Debugging                    | Difficult    | Slow / expensive / Understanding of assembly           |
------------------------------------------------------------------------------------------------------------------
```

### Dynamic analysis methodology
It is always good to have a methodology that you follow and apply. It is not a roadmap that cannot be changed and adapt new paths if needed, but it is a useful starting point for most malware samples, which we will run in the contained environment.

A recommended methodology that is usually used consists of four phases, and includes the four phases:
1. Baseline, where we create the environment with the OS needed, install the tools and take a snapshot we can come back to
2. Pre-Execution, where we perform any necessary configuration, transfer the malware sample to the VM, and start the required tools (monitoring, tracking, and debugging)
3. Post-execution, when we execute the malware, start tracking its behavior and activity (system calls, file access, network traffic), dump/capture screenshots, memory dumps, config files, registry files, unpacked executables...
4. Analysis and documentation, when we analyze and take notes of everything that happened, observe the exhibited behavior, and document events and actions to fill our final report

Often, multiple iterations of phases 2-4 are needed to conclude the analysis.

## Windows processes
Before we start anything related to running a malware, let's dive into the windows system internals and understand the main components of a process and the system we will be doing our investigations and analysis on. This is very important to understand how to track and monitor suspicious processes in your contained environment. We need to understand what's normal first, so we can understand what isn't later.

When reading most of the operating system books, you will see a process is defined as a program in exeution, but it's technically not true, and far more than that. A process is a resource mechanism used to represent a running instance of a program. This resource mechanism is also an object by itself, which is used to manage the resources required by a program in a contained or isolated environment. This means each process has its own environment.

Each process has the following main elements:
- Executable image (the program file), the file that includes the code to be executed. Every process has at least one.
- Private Virtual Address space, where everything the program loads is stored, including the image file, the libraries required, the stack, heap, and other resources. This helps running the process safely without interfering with other processes. The size of this depends on system architecture, process architecture, and the LARGEADDRESSAWARE fkag set to yes or no.
- Private Handle Table, which stores the handles to access resources in the system, like files we want to write to, to interact with the kernel so processes can interact with the system properly. Objects can be many things in a system like registry keys, network sockets, or other processes / Threads
- Access Token, the object that stores the default security context of the process, then grabbed by the thread that actually runs the code. Keep in mind a thread could have a different token later. By default this token includes the security context of the user running the process, plus the groups and permissions of the user.
- Thread(s), the true execution units of a process. They are the ones doing the execution. Each process will have at least one thread and normally will start executing code from the main entrypoint of the executable. Some malware can start executing, however, from TLS (Thread Local Storage). More on that later.

This is how these elements are supposed to be used, however, malware developers may try to exploit weaknesses and cross boundaries, running code in non-conventional ways to avoid detection.

### Process creation steps
Another important execution flow to undersand is how the Windows system creates a process. There are a number of operations that happen in order to create a process, and it is very important to know what they are, their flow, and what happens in each step. Let's go over the process:
1. Check if the file is actually a PE, then open the image file
2. Create and initialize kernel process objects
3. Map image and NTDLL
4. Notify the Windows subsystem process (csrss.exe) (a kernel helper) of a new process and thread
5. Create a Process Environment Block and Thread Environment Block
6. Initialize other process parts needed, like the Heap thread pool
7. Load required DLLs
8. Execute the code's main entrypoint

These will be explored in more detail in coming notes.

## Sysinternals tools
the Sysinternals toolset originally released in 1996 and then was acquired and kept updated by Microsoft. It is a set of utilities for managing, diagnosing, troubleshooting and monitoring a Microsoft Windows environment, but they are also very useful for malware analysts and threat hunters. THe list includes:
- Process explorer, an advanced task manager that could be used to learn about active processes, or what DLLs and handles are opened by them.
- WinObj, which displays the different kernel objects the OS provides and could be accessed by a process
- listDLLs, used to display what DLLs are loaded into a process, similar to the capability of Process Explorer
- Sysinternal Handles tool, a command-line interface to list the handles being used by a certain process
- Procmon, an advanced process monitoring tool that can show, in real-time, what a process or thread activity looks like behind the scenes. It shows events and actions that are happening as the process or thread is active. We'll be using ProcMon in most, if not all, of our basic dynamic analysis. Some useful filters that we can get with procmon are sending or getting data with TCP/UDP, or loading DLLs and executables, creating files, registry activity, or process or thread spawning

Some useful features also available on Procmon include:
- Boot logging, which starts capturing data very early in the booting process, and can be useful to see what activities happen even before user logon, like device driver loading, auto starting services or shell initialization. Even on user logoff or system shutdown
- Drop filtered events reduces the number of captured logs by procmon, as this only stores exactly what we need, using less memory.
- History depth, which stops event capture if the system memory runs too low. We can also configure the count of stored entries.
- Backing files, which stores events in a file automatically in case the system memory runs out so we don't lose that information.

## System processes and services
It's very important to be able to identify what are normal environment processes and what aren't. Once we understand what's running and how, it will be much easier to spot anomalies. This is a threat hunting technique that is also useful here in malware analysis. This is earier with Sysinternals Process Explorer, where we can see the system idle process, memory compression, service control manager, the windows registry (listed, but not truly a process), the System process... you can also notice all process IDs are a multiple of 4. ID 4 always belongs to System. Remember to check your system to know what is considered normal behavior to get a baseline.

It is also important to know where the system images are in case we need to investigate them for legitimacy. 64-bit executables are at C:\Windows\System32, while 32-bit executables that run on the 64-bit program (Through Windows on Windows 64-bit, WoW64) are on C:\Windows\SysWoW64. This redundancy exists because 32-bit processes cannot load 64-bit libraries, and vice-versa, they cannot work without this compatibility emulation. The WoW executables load both the 32 and 64 bit NTDLL entries, as they send requests to the 32-bit one and forward it to the 64-bit library.

To make it easier for file exploration depending on architecture, when you open those directories, Windows actually does a logical path redirection to take you to the proper folder in the file explorer you open, so the directories are actually logical links. Same happens with the registry, the application that accesses the registry gets a logical version of its architecture-specific registry displayed to it. This is important in case you open a 32-bit malware sample in a 64-bit system and it seems to be loading libraries from unexpected locations.

There are some important windows internal files to keep track of, mainly NTDLL.dll which contains internal support functions and system-service helpers, mapping incoming Windows API requests to their corresponding kernel services, and Kernel32.dll, which is a user-mode DLL that passes user requests aimed to the kernel to NTDLL.

## Injection techniques
Process injection is a technique used by malware as an evasion mechanism. It involves executing custom code within another process's address space to be stealthy, and in some cases, to achieve persistance. Process injection can be achieved through several means. We'll go over the common ones, how they work, and what methods do they use. Understanding these techniques facilitates the process of malware analysis and reverse engineering, assisting in the dection and defense against them.

### Classical DLL injection
Here, the malware writes the path to its DLL into the address space of the legitimate process and creates a remote thread in the targeted process to guarantee the remote process will load the injected DLL, and thus the malicious code.

To achieve this, first the malware selects a target process to injectis is code, for example `svchost.exe`. Usually the malware searches for a target process using three APIs:
- **CreateToolhelp32Snapshot**, used to retrieve a snapshot of the heap or module state of a particular process, or for all processes
- **Process32First**, which returns information about the first process in the snapshot
- **Process32Next**, which we can use to iterate over the list of processes if we retrieve them all

The malware will load the malicious DLL into the target process by using `OpenProcess` to get a handle of the target process, and use it to interact with the target process, allocating memory space with `VirtualAllocEx` and calling `WriteProcessMemory` to write the path of the malware DLL in the target process to load it in. Then, the malware will call an API function such as `CreateRemoteThread`, `RtlCreateUserThread` or `NtCreateThreadEx` to execute the injected code on the target process. This is done by passingg the `LoadLibrary` address to any of these APIs to ensure the injected DLL will be executed by a remote process on behalf of the malware.

Due to this abuse being so pervasive, several security products track usage and flag potential executions of `CreateRemoteThread`. This kind of injection also needs the malicious DLL to exist on disk, which is something that leaves a trace and can be detected and restored by a forensics analyst. Sophisticated attackers will not use this technique.

### PE injection
In this injection technique, the malware **copies** its malicious code (instead of passing it as an address to `LoadLibrary`) into an existing process and runs it using shellcode or by calling `CreateRemoteThread`. The advantage of this technique over the previous one os that it does not require the malicious DLL on the disk. The process is similar to the classical DLL injection, but in this case, the allocated memory stores the entire malicious DLL code, and not a reference to it on disk.

The problem with this technique is that the base address of the copied image will be changed. Injecting the malware PE into another process will generate a new base address and this is unpredictable, which enforce it to dynamically recompute the fixet addresses of its PE. To solve this problem, the malware should find the address of its relocation table in the host process, and then resolve the absolute addresses of the copied image by iterating the reloaction descriptors.

The above essentially means that writing the DLL into the host process will change the address we need to execute it on, which is a problem because then the malware doesn't know where to tell the process to start reading the malicious DLL, so it needs to re-calculate that location. After calculating this, the malware will send `CreateRemoteThread` the DLL starting address to run this malware.

This technique is less stealthy than other techniques, such as memory module or reflective DLL injection, since it relies on extra Windows APIs, such as `LoadLibrary` and `CreateRemoteThread`. Reflective DLL injection creates a DLL that can map itself onto memory for execution instead of relying on the Windows loader, and in the Memory Module technique the loader or injector is resoonsible for mapping the malicious DLL to memory instead of the DLL mapping itself.

When analyzing PE injection, you may see nested loops (two `for` loops) before a call to the `CreateRemoteThread` function. This technique is used by Crypters, the type of sofrware used to obfuscate malware traces using encryption.

### Process hollowing, process replacement or RunPE
In this technique, the malware unmaps the code of a legitimate process from memory and overwrites the memory space of the victim process with its malicious code. THe malware starts by creating a new process in a **suspended** state. To do this, the malware calls `CreateProcess` and sets a flag of process creation into `CREATE_SUSPENDED` (`0x00000004`). The primary thread of the newly created process will remain suspended until the malware calls the `ResumeThread` function. 

Then, the malware replaces the legitimate file contents of the victim thread with its malicious payload. This memory unmapping is done using the `NtUnmapViewOfSection` or `ZwUnmapViewOfSection` APIs, which release all memory contents when pointed to a section. Next, the loader calls `VirtualAllocEx` to allocate new memory space for the malware. The loader uses the `WriteProcessMemory` function to write the malware's sections into the target process's memory space.

The malware uses the `SetThreadContext` function to set the entrypoint of the new code section written to the target process, and the suspended thread is resumed using `ResumeThread`. THis means that the loader thread we suspended has no malicious code in it, but it is rather reading and executing that code inside of a legitimate thread, which makes it harder to detect and analyse.

### Thread execution hijacking (SIR: Suspend, Inject, Resume)
This technique, instead of injecting the shell code using the DLL nade, injects malicious code into the existing thread of a process (avoiding the overhead of process and thread creation) and then uses the target to start a thread in itself. Hence during analysis we may find the functions `CreateToolhelp32Snapshot`, `Process32First` and `OpenThread`. THe malware will call `SuspendThread` to suspend a victim thread from execution, add the malicious code with `VirtualAllocEx` and `WriteProcessMemory`, then resume the thread.

The process is very similar to classical DLL injection, however, this technique is more sophisticated and more stealthy, as it involves hijacking an existing thread instead of creating a new one, which could raise alarms in security monitoring software. The SIR approach may arise a problem for adversaries, since suspending a process and resuming it during a system call may crash the system. This is why this technique is usually seen in more sophisticated attackers.

### Hook injection (SetWindowsHookEx)
Hooking techniques are used to intercept function calls. Malware can use it to load their malicious DLL upon an action getting triggered in a particular thread.

In order to install a hook function into the hook chain, malware calls `SetWindowsHookEx`. This function requires four arguments:
1. The event type, which indicates the range of the hook type and varies from mouse inputs, to keyboard pressing keys, for example, or opening new windows or openingg applications. Essentially, system events.
2. A pointer to a function the malware wants to invoke when the event is called by the hook
3. The module handle which contains the function the malware will invoke. This is typically the module that is calling `SetWindowsHookEx` function to create a hook
4. The thread that will be associated with the hook procedure. If the value is set to zero, all threads will perform the action upon the event triggering. Commonly malware only targets one thread to minimise noise, so it is possible you may see thread searching (`CreateToolHelp32Snapshot`, `Thread32Next`) before running hook creation.

### Injection via Registry Modifications
Some registry keys are used by malware for both code injection and persistance, mainly `AppCertDlls`, `Appinit_DLL` and `IFEO` (Image File Execution Options).

These keys can be found here:
```
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Windows\Appinit_Dlls
HKLM\Software\Wow6432Node\Microsoft\Windows NT\CurrentVersion\Windows\Appinit_Dlls
HKLM\System\CurrentControlSet\Control\Session Manager\AppCertDlls
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Windows\Image File Execution Options
```

Malware can use `Appinit_DLL` key to store the location of their malicious library, so another process will load their library. All libraries stored under this key will be loaded into every process that loads the `User32.dll` library. This is a very common library used to store graphical elements, like dialob goxes. Therefore by modifying this subkey, the malware guarantees that most of the processes will load their malicious DLL.

`AppCertDlls` has a very similar operation to AppInit, but the libraries under that key will be loaded whenever someone calls the following Win32 API functions:
- `CreateProcess`
- `CreateProcessWithLogonW`
- `CreateProcessAsUser`
- `WinExec`
- `CreateProcessWithTokenW`

Finally, `IFEO` is typically used for debugging purposes. For example, the debugger value under this subkey is used by developers to attach a program to another executable for debugging. So, if an executable is launched, the program attached to it will also be launched. To run this feature we can attach the path of the debugger to the executable that we wish to analyze. Malware can change this registry key to inject its payload into a target executable.

### APC Injection and AtomBombing
Malware exploits Asynchronous Procedure Calls (`APC`) to get legitimate threads to execute their malicious code. This can be done by attaching their code to the APC queue of the target thread. Every thread is associated with a queue of APCs that wait for execution when the thread calls execution of asynchronous tasks on that queue. `AtomBombing` also depends on APC injection, but it uses the atom table to write into the memory space of another thread. This is called the thread being in an alterable state, which means it is waiting for a kernel-level APC to be delivered to it. Note that not all threads are in an alterable state at all times. 

The function calls below can cause a thread to enter an alterable state:
- `SleepEx`
- `WaitForSingleObjectEx`
- `SignalObjectAndWait`
- `MsgWaitForMultipleObjectsEx`
- `WaitForMultipleObjectsEx`

The way this attack is executed is the malware finds a thread in an alterable state and calls `OpenThread` and `QueueUserAPC` to queue an APC to the selected thread. `QueueUserAPC` requires three arguments:
- A handle to the selected target thread in an alterable state
- A pointer to the function which the malware wants to execute
- The parameter which will be pased to the function pointer

### Extra Window Memory Injection (EWMI)
Some malware families like PowerLoader use EWMI injection as a technique, which relies on injecting code into "Explorer tray window"'s extra window memory. Upon the creation of a window, the application can determine an additional number of memory bytes, called Extra Window Memory, where code can potentially be added. Since the EWM space is limited, the malware writes the code in a shared section of `explorer.exe`, and then uses `SetWindowLong` and `SendNotifyMessage` to make a function pointer to the shellcode, and finally executes it from the original window thread.

Malware can either create a shared section and have it mapped both to itself and another process, or open an already existing shared section. The second method is used more often since the first one incurs an overhead while allocating heap space and several potentially surveilled APIs, like `NTMapViewOfSection`. After writing the code into the shared section, the malware uses `GetWindowLong` and `SetWindowLong` to enter and change additional window memory of `Shell_TrayWnd`, accessible by all other programs if needed, pointing the extra memory to the attacker's shellcode.

After that modification, the malware calls `SendNotifyMessage` to trigger the code execution, because when it is executed, `Shell_TrayWnd` will receive and transfer the control to the adderss indicated on the extra memory value set earlier.

### Early Bird API Injection
Although simple, early bird API injection is a powerful technique that enables adversaries to inject malicious code into target processes before their main thread starts. It cane vade detection by most anti-malware products, which use windows hook engines. This injection was used, for example, by the TurnedUp backdoor, and the DorkBot malware.

The steps to execute Early Bird API injection are as follow:
1. Create a legitimate process like `svchost.exe`, in a suspended state
2. Allocate memory to the created process
3. Write the shellcode into the allocated memory space
4. Queue an Asynchronous Procedure Call to the main thread of the created process
5. Call the `NtTestAlert` function to instruct the kernel to execute the malicious code when the main thread resumes running, as `NtTestAlert` will put the thread in an alertable state

### Hooking techniques
Adversaries use API hooking (based on process injection) to intercept calls to Windows APIs in order to modify the input or output of these commands. Use cases are stealing passwords, preventing security tools from loading, hijacking network connections, keystroke logging, and hiding processes and files. Using API hooking, an attacker can have full control over a particular process and record user data resulted from the interaction with the process, like browsers, visited websites, antivirus programs and files scanned by these programs. The attacker also gets the ability to capture sensitive data stored in the process memory or API arguments.

API hooks are different from hook injection in the sense that hook injection will replace the hook being called by the thread to call the malware's malicious code, while API hooking will replace the Windows API's function so it can monitor what is being sent to legitimate system hooks. For example, modifying the `CreateFileW` function so the malware can monitor file creation

This technique is used for a lot of evasion protocols on malware, like hiding itself from antivirus applications by removing itself from the process list, its files from the file enumeration APIs, or its registry keys. They also protect themselves from being detected on banking stealing by hooking and hiding HTTP and networking connections, or Firefox APIs for when the malware modifies a banking page.

API hooking uses several techniques, of which the most common are:
- Inline API hooking, which changes process flow through hotpatching or modification of code during runtime, changing generally the first five bytes of assembly with `jmp [hooking_function]`, which modifies the function call, then `jmp`'s back to the original function 
- Inline API hooking with Trampoline, which instead of modifying the API call request, the return values and data is modified. THis gives the malware more control of the API's output, altering the values before returning them, for example adding JavaScript code to hijack credentials
- IAT hooking, while not widely used, is also a present technique. It involves modifying the Import Address Table so when the program tries to run an imported function, it jumps to the malicious hooking function instead of the actual API. This attack is useful for programs using an import table, but not for ones using dynamically loading APIs.

Detecting API hooking can be achieved through memory forensics, but in general:
- Detecting API hooking is based on process injection, so detecting the two is similar
- Tools exist for this purpose, like `malfind`, `hollowfind` or some `volatility` commands, such as `apihooks`, which can scan process libraries to search for hooked APIs (which will generally start with `jmp` or a `call` after being hooked)
- `vaddump` can also be used to dump a memory address and then use static analysis tools on it such as IDA Pro. Then, that found shellcode can be disassembled to understand the motivation of that API hooking.

## Persistent methods
Here we'll go over the most common methods used to achieve persistency on a system. There are more, but these are the most commonly used.

### Windows registry
While the windows registry is used by the system to store and retrieve system settings, malware authors additionally use it for persistence of their programs across system reboots. While there are many locations in the registry that are used to autorun malicious executables, these are the most commonly used ones:
```
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Run
HKCU\Software\Wow6432Node\Microsoft\Windows NT\CurrentVersion\Windows
HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce\*
HKLM\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Run\*
HKLM\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\RunOnce\*
HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer\Run\*
```

There is software to help you automatically check these locations, like `Autoruns`. While monitoring or analyzing malware samples, regardless of the method you use, make sure you look out for windows APIs that reference the registry and especially those that do additions or modifications. Those could be an indicator of persistance being attempted.

### Task scheduling
Malware authors might alsdo use the task scheduler to schedule their malware to start at a certain time, after a certain event, or while the system boots. One of the easiest ways to schedule a task is using the `schtasks` command, like this example that configures a scheduled task to run a dropped malware executable on boot:

```
schtasks /create /tn svchost /tr C:\Users\[Victim user]\AppData\Local\[Name]\svchost.exe /sc ONSTART /f
```

We can detect the creation of scheduled tasks by monitoringg execution of the `schtasks` command using `Autoruns` by SysInternals, or checking for any tasks under `C:\Windows\System32\Tasks`. There we can find the tasks' XML configuration files. Keep in mind, scheduled tasks can be created through WMI or PowerShell as well.

### Startup folders
Another easy way to run malware and achieve persistance is using the Windows Startup Folders by having the malicious programs stored there. The malware will run once the user logs into the system. These folders are divided between **user-level** and **system-level** directories. If a program was placed into a user folder, it will only work when that user logs in. Malware on the system folder will run for any user logs in.

One of the easiest ways to find the folder and what's in it is by opening a Windows Run dialog (win + r) and typing `shell:startup`. THer locations are `C:\ProgramData\Microsoft\Windows\Start Menu\ Programs\ StartUp` for all users, and `C:\Users\[Username]\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup` for each specific user.

### WinLogon
The `WinLogon` process is responsible for:
1. User logon and logoff
2. Startup and Shutdown
3. Locking the screen

Malware authors can modify the registry entries used by WinLogon to achieve persistance. The Winlogon process will, by default, launch the `userinit.exe` program that is responsible for running different logon scripts and network related tasks. Userinit.exe is controlled by a registry location for Winlogon. This userinit, in turn, is responsible for running `explorer.exe`, controlled by a Userinit registry key tht sets the shell.

The registry keys are:
```
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\Userinit
```

Persistance here is achieved by adding more executables to the Shell registry entry in Winlogon. Again, there are other locations, so using `SysInternals Autoruns` can be useful to see all of them at once.

### AppInit_DLLs
As metioned earlier, AppInit_DLLs is aregistry value that controls what DLLs are loaded into a process that loads the User32.DLL library. Now, since many processes might use that library, it means that whatever DLLs are specified in this registry value will be loaded by them to achieve persistance. These live in two registry locations:
```
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Windows
HKLM\Software\Wow6432Node\Microsoft\Windows NT\CurrentVersion\Windows
```

Based on Microsoft's documentation, this value is disabled by default in Windows 8 onwards if the system is using secure boot, so modern systems may not present this type of persistance mechanism.

### DLL Search Order hijacking
By now we know that programs require libraries in order to achieve functionalities like interacting with the system. Those DLLs are what sometimes comes shipped with the application, or they could be system-level DLLs that the application uses. Whenever a program needs to search a library, Windows has an order of locations it can search those DLLs in, like the path variable on linux. Malware authors can hijack this search order to achieve different objectives, including persistance. The order of checking is:

1. DLLs already loaded in memory
2. DLLs defined in the KnownDLLs variable, which can gbe found on registry key `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs`
3. The directory from where the application is loaded
4. The system directory `C:\Windows\System32`
5. The 16-bit system directory `C:\Windows\System`
6. The Windows directory `C:\Windows`
7. The current working directory
8. The directories defined by the `PATH` environment variable

### Windows Service
We know that a service is an application running in the system background, and usually users don't directly interact with it. Malware developers could also configure and install their malware to run in the system's background as a service and achieve persistance. This could be accomplished by running the malware as a standalone executable, as a DLL loaded into a container (`svchost.exe`), or as a kernel driver service.

Windows serfvices can be controlled via CLI using the `sc` command or using the `services.msc` GUI application. This is an example that could be used directly in the command line to create a service named Skype, have it auto-start on boot, and automatically run our binary:

```
sc create Skype binPath=C:\Users\IEUser\AppData\Local\Skype\skype.exe start= auto && sc start Skype
```

We can see that a service is running by using `services.msc` or by seeing it in the terminal with `sc query [service name]`. Windows defines services under the registry location `HKLM\SYSTEM\CurrentControlSet\Services` where each service has its own subkey with different settings.

Malware developers generally use the `svchost.exe` container to run their payloads, especially DLL malware. Each `svchost.exe` is responsible for a group of services. This is how Microsoft makes different tasks related to maintenance and management much easier. The groups are defined in the reggistry key `KHLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Svchost`, where a registry entry holds the names of all services it controls. Then, each service under `HKLM\SYSTEM\CurrentControlSet\Services` has an imagePath that attaches it back to its svchost. For example, the `Schedule` service is part of the `netsvcs` group, so its image key looks like this:
```
%SystemRoot%\System32\svchost.exe -k netsvcs
```

Services should contain a `Parameter` value and a `ServiceDLL` value, which is what will be loaded on the image file. This is what malware modifies to achieve peristance. We can also check what is being loaded by svchost.exe by using the CLI tool `tasklist.exe`, like:
```
tasklist /svc /fi "imagename eq svchost.exe"
```

Or, alternatively, the **Services** tab in **Process Hacker**, which has the additional benefit of allowing you to double-click on a service and loading additional information about that specific service, like its DLL, user account running it, or its scheduling group.

## DLL Analysis
If we gather a malicious DLL to run analysis on, how do we do that? Especially since a lot of DLLs need a parent program to run properly to use its exported functions. You could write an application for this, but Windows already comes with a tool for this: `rundll32.exe`. It can be used the following way:

```
rundll32.exe [DLLName],[ExportedFunction],[Arguments]
```

That said, there are better tools to do this, which we'll explore on the next section.

## Tools and automation
Here we'll go over some very useful tools to use during analysis. There are a lot more that are useful, but these are great to start with:
- **ProcDOT** to visualize process activity
- **Fiddler** to intercept network traffic
- **DependencyWalker**
- **RegShot**
- **Process Hacker**

Apart from these, there are a lot of other very useful ones, like the SysInternal toolset, online scanners like Cuckoo, any.run or VirusTotal.

### ProcDOT
By now you've probably tried Process Monitor. You should also have an idea of what traffic capture is. ProcDOT combines both of these worlds and displays the results for you. It helps in visualizing process activity, combining its interaction with the system with its access to networks. YOu can load it with a procmon CSV file that was taken running the sample and a pcap file, and let procDOT generate the interaction graphics.

### Fiddler
Fiddler is a web debugging proxy to log all HTTP(S) traffic between your computer and the internet. Inspec traffic, set breakpoints, and fiddle with the request and response. We can use this tool to intercept and analyze the malware's HTTP(S) communications, for example, with a C2 server.

### DependencyWalker
This is a very useful tool to scan Windows executables and build a hierarchy of the libraries and functions that are being referenced by the executable. It can list all the modules exported by a library and the call functions that are actually being called by other libraries.

### RegShot
RegShot is an open source tool that lets us compare between two registry snapshots. This is is useful to check what registry keys might a malware have changed on your system after execution.

### Process Hacker
Process Hacker is a free, powerful, multi-purpose tool that helps you monitor system resources, debug software and detect malware. Process Hacker is similar to Process Explorer in its capabilities, but adds some powerful features on top of that, for example:
- string searching
- memory region dumping
- shellcode extraction

## Windows APIs
Windows Application Programming Interface (WinAPI) calls are defined as a set of functions and data structures used by Windows programs to perform some operating system functionality through the system. Generally, Windows program operations involve calling several API functions, for example, you don't want to implement writing to disk yourself, so you can just ask a WinAPI function to write a file for you.

These APIs by themselves are not necessarily malicious, the way they are used might be an indicator of malware running on the system.
