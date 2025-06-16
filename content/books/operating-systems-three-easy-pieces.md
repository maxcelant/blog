---
title: Operating Systems, Three Easy Pieces
---
## Introduction to Operating Systems
Each process accesses its own private virtual address space (sometimes just called its address space), which the OS somehow maps onto the physical memory of the machine.

So now you have some idea of what an OS actually does: it takes physical resources, such as a CPU, memory, or disk, and **virtualizes** them. It handles tough and tricky issues related to **concurrency**. And it stores files **persistently**, thus making them safe over the long-term.

When a system call is triggered, we enter kernel mode, which has heightened control of the system and can do things like IO, once that call is over, we return to user mode.

## Part I: Virtualization
Question: How does an OS allow us to run hundreds or more processes at once? 

**Time sharing** is that idea that the CPU will give each process some time to run. 

**Policies** are algorithms to perform some kind of decision by the OS. Like "which process should I run next?". A scheduling policy is used to determine this. Looks at historical data, workload knowledge and performance metrics.
###### How do we execute a process?
The executable is loaded into memory from disk. It allocates some memory to the stack and the heap. It then opens three file descriptors for STDIN, STDOUT, and STDERR.

The three states of a process are running, ready, blocked. Blocked means another process can step in. A process can be moved between the ready and running states at the discretion of the OS.

The register context will hold, for a stopped process, the contents of its register state. When a process is stopped, its register state will be saved to this memory location; by restoring these registers (i.e., placing their values back into the actual physical registers), the OS can resume running the process.

When finished, the parent will make one final call (e.g., `wait()`) to wait for the completion of the child, and to also indicate to the OS that it can clean up any relevant data structures that referred to the now-extinct process.

Sometimes people refer to the individual structure that stores information about a process as a **Process Control Block (PCB)**.
##### Interlude: Process API
`fork()` spawns a new process from the position of the command. The execution of the child and parent is non-deterministic. Adding `wait()` will cause the parent to wait for the child process to finish before executing.

`exec()` does not create a new process; rather, it transforms the currently running program (formerly p3) into a different running program (wc). It does not return to the original program after terminating.

The separation of `fork()` and `exec()` is essential in building a UNIX shell, because it lets the shell run code after the call to `fork()` but before the call to `exec()`; this code can alter the environment of the about-to-be-run program, and thus enables a variety of interesting features to be readily built.

When you close a file descriptor, and you open new one, it'll take that first available slot, meaning the slot that you just closed. This allows it to redirect standard out/in/err to something like a file.
##### Mechanism: Limited Direct Execution 
When the OS wishes to start a program running, it creates a process entry for it in a process list, allocates some memory pages for it, loads the program code into memory (from disk), locates its entry point (i.e., the `main()` routine or something similar), jumps to it, and starts running the user’s code.

When a program is in **user mode**, it does not have many privileges, it can't do I/O operations, etc. However, we can enter **kernel mode** which has these heightened privileges. We enter it using a special **trap** instruction, and once the kernel does what it needs to do, there is a **return from trap** back to user mode.

C system calls are handwritten in Assembly. They need to carefully handle the process arguments, return values and the hardware specific trap instruction.

When a trap instruction is triggered, important process information is pushed onto the kernel stack (like program counter, flags, etc) for the duration of the kernel mode, once return-from-trap is called, those get popped off the stack.

The OS sets up a **trap table** at boot time and tells the CPU its location so that it can tell the hardware which code to run when a specific trap is called using a **trap handler**, which are just special instructions. 

Think of the trap table as a key-value store where the key is a the trap number and the value is a function pointer to the trap handler.

Programs are interrupted intermittently so that the OS can regain control and do what it wants, like give control to another process.

Each process gets its own kernel stack. When context switching, we save critical data of one process into its kernel stack, then move the CPU stack pointer the new processes kernel stack.



