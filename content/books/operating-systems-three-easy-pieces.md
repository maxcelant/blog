---
title: Operating Systems, Three Easy Pieces
---
## Introduction to Operating Systems
Each process accesses its own private virtual address space (sometimes just called its address space), which the OS somehow maps onto the physical memory of the machine.

So now you have some idea of what an OS actually does: it takes physical resources, such as a CPU, memory, or disk, and **virtualizes** them. It handles tough and tricky issues related to **concurrency**. And it stores files **persistently**, thus making them safe over the long-term.

When a system call is triggered, we enter kernel mode, which has heightened control of the system and can do things like I/O, once that call is over, we return to user mode.

## Part I: Virtualization
>[!question]
> How does an OS allow us to run hundreds or more processes at once? 

**Time sharing** is that idea that the CPU will give each process some time to run. 

**Policies** are algorithms to perform some kind of decision by the OS. Like "which process should I run next?". A scheduling policy is used to determine this. Looks at historical data, workload knowledge and performance metrics.
###### How do we execute a process?
The executable is loaded into memory from disk. It allocates some memory to the stack and the heap. It then opens three file descriptors for STDIN, STDOUT, and STDERR.

The three states of a process are running, ready, blocked. Blocked means another process can step in. A process can be moved between the ready and running states at the discretion of the OS.

The register context will hold, for a stopped process, the contents of its register state. When a process is stopped, its register state will be saved to this memory location; by restoring these registers (i.e., placing their values back into the actual physical registers), the OS can resume running the process.

When finished, the parent will make one final call (e.g., `wait()`) to wait for the completion of the child, and to also indicate to the OS that it can clean up any relevant data structures that referred to the now-extinct process.

Sometimes people refer to the individual structure that stores information about a process as a **Process Control Block (PCB)**.
##### Interlude: Process API
`fork()` spawns a new process from the position of the command. The execution of the child and parent is non-deterministic. Adding `wait()` to the mix will cause the parent to wait for the child process to finish before executing.

`exec()` does not create a new process; rather, it transforms the currently running program into a different running program. It does not return to the original program after terminating.

The separation of `fork()` and `exec()` is essential in building a UNIX shell, because it lets the shell run code after the call to `fork()` but before the call to `exec()`; this code can alter the environment of the about-to-be-run program, and thus enables a variety of interesting features to be readily built.

When you close a file descriptor, and you open new one, it'll take that first available slot, meaning the slot that you just closed. This allows it to redirect standard out/in/err to something like a file.
#### Mechanism: Limited Direct Execution 
When the OS wishes to start a program running, it creates a process entry for it in a process list, allocates some memory pages for it, loads the program code into memory (from disk), locates its entry point (i.e., the `main()` routine or something similar), jumps to it, and starts running the user’s code.

When a program is in **user mode**, it does not have many privileges, it can't do I/O operations, etc. However, we can enter **kernel mode** which has these heightened privileges. We enter it using a special **trap** instruction, and once the kernel does what it needs to do, there is a **return from trap** back to user mode.

C system calls are handwritten in Assembly. They need to carefully handle the process arguments, return values and the hardware specific trap instruction.

When a trap instruction is triggered, important process information is pushed onto the kernel stack (like program counter, flags, etc) for the duration of the kernel mode, once return-from-trap is called, those get popped off the stack.

The OS sets up a **trap table** at boot time and tells the CPU its location so that it can tell the hardware which code to run when a specific trap is called using a **trap handler**, which are just special instructions. 

> [!note]
> We can think of what the OS is doing as "baby-proofing" the room to ensure the user doesn't do anything it shouldn't.

Think of the trap table as a key-value store where the key is a the trap number and the value is a function pointer to the trap handler.

Programs are interrupted intermittently so that the OS can regain control and do what it wants, like give control to another process. Each process gets its own kernel stack. When context switching, we save critical data of one process into its kernel stack, then move the CPU stack pointer the new processes kernel stack. 

The `switch()` routine (in Assembly) will store A's registers into its process structure and load B's registers from its structure then move the stack pointer to point to B's kernel stack.

The interrupt to regain control used to be cooperative, meaning the OS would regain control when the programmed went into kernel mode, but that ended up being lackluster. Instead, the OS uses a timer to regain control every so often. 

#### Scheduling: Introduction 
Scheduling policies are sometimes called **disciplines**. The collective of processes running on a system is called a **workload**. 

**Turnaround time** is a performance metric that basically tells us how long a job (process) ran for. 

**Response time** is the time from when a job enters the system to when its first scheduled.

Response time and Turn around time are a trade-off, you can't really have both with a simple scheduler. You either share the CPU or you run a job to completion.

**Convoy effect**, where a number of relatively-short potential consumers of a resource get queued behind a heavyweight resource consumer. 

Preemption is the idea that the scheduler can context switch.

There are some scheduling algorithms (in increasing complexity):
- First In First Out
- Shortest Job First
- Shortest Time-to-Completion First (Preemptive)
	- When a new job enters, it determines which has shortest turnaround time and runs that to completion.
- Round-Robin (Time Slicing) - Make window large enough to amortize context switch.

Operating systems are smart and will fill the space of process blocking I/O by running a separate process.

#### Scheduling: Multi-Level Feedback Queue
Uses past behavior to predict the future. The algorithm uses a series of queues with different priorities. The jobs on the highest priority queue will be run, but keep in mind that the priority of the jobs are constantly evolving. 

There are a set of **rules** that will cause a job to move up or down the ladder. For example, when a job enters the queue, it starts at the highest priority. If the job is switched back to numerous times and continues to run, it'll decrement in priority each time. This is telling the kernel that this is a long term job, and not a short bursty one. 

If a job releases the CPU before its time slice is up, then the kernel will keep it at the same priority level. This usually occurs with heavy I/O / interactive jobs. We want those to continue to have priority. However, this could create a problem where long running jobs are starved by fast ones. So there's a rule that periodically make all jobs the highest priority.

To avoid gaming the scheduler, once a job has reached its time slice quota, it's demoted a step in the ladder. 

#### Scheduling: Proportional Share
Guarantees that each job gets a certain amount of CPU time. Uses a ticket based system (also called **lottery scheduling**), which basically means the more tickets you have, the higher chance you have of getting the CPU.

The algorithm is very simple. Have a `total`value. Pick random `winner` value. Walk the list of jobs (that have a certain currency), accumulate the current jobs value to the `total`, and stop once the total exceeds the `winner` value.

**stride scheduling** achieves proportional time sharing a predictable way by always running the job with the lowest pass value of the bunch and incrementing it by its stride value once it runs. 

#### Multiprocessor Scheduling
Talking about CPU caches, there's **temporal locality** (likelihood you'll use a piece of data again) and **spatial locality** (likelihood a piece of data near your data will be used).

**Cache affinity** is the idea that it's advantageous to run a program on the same CPU, because a lot of the programs state will already have been loaded onto that CPU's caches. 

**Multi-Queue Scheduling** avoids the locking single queue of **SQMS**, because each CPU will have its own queue of jobs. Work stealing is the idea of one queue peeking another queue and stealing jobs if there are more in the other one. This avoids job imbalances.

#### The Abstraction: Address Spaces
To recap, the goal of virtualizing memory is to isolate each program's memory, and give a simple abstraction for the running program so that it thinks it has a big empty open field for its data and code.

**Address space** is a running programs view of the memory of the system. It's an abstraction to simplify a programs memory from the physical memory of the system. It's made up of three parts: code, stack and heap. A program's code might be the first 1KB, and the heap will grow downwards from there while the stack starts at 16KB and will grow upwards.

#### Interlude: Memory API
`malloc()` takes a `size_t` and returns a pointer to that newly allocated space on the heap. `malloc()` returns a `void`, which is why we have to type cast it to the type of pointer we want. 

```c
int* x = malloc(10 * sizeof(int));
printf("%d\n", sizeof(x));
```

`free()` takes a pointer assigned from `malloc()` and frees the heap memory. Lots of languages do this all for you with the help of a **garbage collector**.

It's common to forget to allocate memory entirely or put just too little, which results in a segmentation fault. 

`malloc` is a library wrapper around the `brk` and `sbrk` syscalls which move the programs **break** (the programs end of heap).

#### Mechanism: Address Translation
**Address translation** is when the hardware transforms a virtual address into the physical one. Although the process might think it's at address space 0, in reality the OS sets a **base** for that program, which is the actual physical address that it will be translated to. The **bound** is the upper limit of the a virtual address space. It's used to ensure that the memory accessed is still within the bounds of the programs memory. 

>[!info] 
>The **Memory Management Unit (MMU)** is a physical unit that exists on a CPU. 

The problem with base-and-bound is that it doesn't utilize the unused memory space between the stack and heap of a program well, this is called **internal fragmentation**.
#### Segmentation 
Allows us to split the stack, heap, and code **segments** in different non-contiguous memory addresses. This fixes the wasteful memory problem with standard base-and-bound, but we now need to keep track of 3 base and size values. Each program would have its segment values stored, so that on a context switch, they can be safely swapped out. 

There exists a **segment table** which denotes the index, and base/ bound of each segment we care about (code, stack, heap). When we look at a virtual address, the first two bits correspond to the segment index ("Am I looking at a stack segment?"), and the remaining 12 bits are the offset of that address from that segments base address.

```python 
seg_idx = virtual_addr[:2]
offset = virtual_addr[2:]
if offset > bounds[seg_idx]:
    raise Exception()
else:
    phy_addr = base[seg_idx] + offset
    register = access_mem(phy_addr)
```

Since the stack grows in the opposite direction, we need to add a bit to our segment table to determine whether our segment is growing in the positive or negative direction.

