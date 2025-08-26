---
title: Operating Systems, Three Easy Pieces
---
## Introduction to Operating Systems
Each process accesses its own private virtual address space (sometimes just called its address space), which the OS somehow maps onto the physical memory of the machine.

>[!quote]
>So now you have some idea of what an OS actually does: it takes physical resources, such as a CPU, memory, or disk, and **virtualizes** them. It handles tough and tricky issues related to **concurrency**. And it stores files **persistently**, thus making them safe over the long-term.

When a system call is triggered, we enter kernel mode, which has heightened control of the system and can do things like I/O, once that call is over, we return to user mode.

## Part I: Virtualization

**Time sharing** is that idea that the CPU will give each process some time to run. 

**Policies** are algorithms to perform some kind of decision by the OS. Like "which process should I run next?". A scheduling policy is used to determine this. Looks at historical data, workload knowledge and performance metrics.
###### How do we execute a process?
The executable is loaded into memory from disk. It allocates some memory to the stack and the heap. It then opens three file descriptors for STDIN, STDOUT, and STDERR.

The three states of a process are running, ready, blocked. Blocked means another process can step in. A process can be moved between the ready and running states at the discretion of the OS.

The register context will hold, for a stopped process, the contents of its register state. When a process is stopped, its register state will be saved to this memory location; by restoring these registers (i.e., placing their values back into the actual physical registers), the OS can resume running the process.

When finished, the parent will make one final call (e.g., `wait()`) to wait for the completion of the child, and to also indicate to the OS that it can clean up any relevant data structures that referred to the now-extinct process.

Sometimes people refer to the individual structure that stores information about a process as a **Process Control Block (PCB)**.
#### Interlude: Process API
`fork()` spawns a new process from the position of the command. The execution of the child and parent is non-deterministic. Adding `wait()` to the mix will cause the parent to wait for the child process to finish before executing.

`exec()` does not create a new process; rather, it transforms the currently running program into a different running program. It does not return to the original program after terminating.

The separation of `fork()` and `exec()` is essential in building a UNIX shell, because it lets the shell run code after the call to `fork()` but before the call to `exec()`; this code can alter the environment of the about-to-be-run program, and thus enables a variety of interesting features to be readily built.

When you close a file descriptor, and you open new one, it'll take that first available slot, meaning the slot that you just closed. This allows it to redirect standard out/in/err to something like a file.
#### Mechanism: Limited Direct Execution 
When the OS wishes to start a program running, it creates a process entry for it in a process list, allocates some memory pages for it, loads the program code into memory (from disk), locates its entry point (i.e., the `main()` routine or something similar), jumps to it, and starts running the user’s code.

When a program is in **user mode**, it does not have many privileges, it can't do I/O operations, etc. However, we can enter **kernel mode** which has these heightened privileges. We enter it using a special **trap** instruction, and once the kernel does what it needs to do, there is a **return from trap** back to user mode.

C system calls are handwritten in Assembly. They need to carefully handle the process arguments, return values and the hardware specific trap instruction.

When a trap instruction is triggered, important process information is pushed onto the kernel stack (like program counter, flags, etc) for the duration of the kernel mode, once return-from-trap is called, those get popped off the stack.

>[!tip] 
>Think of it like a trapdoor to another control mode. 

The OS sets up a **trap table** at boot time and tells the CPU its location so that it can tell the hardware which code to run when a specific trap is called using a **trap handler**, which are just special instructions. 

> [!tip]
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

#### Free Space Management 
A **free list** is a term used to describe a data structure used to keep track of free allocatable space. For now, we can think of it as a linked list, where each node is a block of free space. **Splitting** is the act of taking available space a cutting it so that the allocated chunk is removed from the free space.

**Coalescing** is a mechanism used by allocators to merge newly freed member blocks with nearby neighboring free space to make a larger free block. 

**Allocators** are the components responsible for allocating and de-allocating memory in the most optimized fashion.

A **header** is allocated before the chunk the user wants so that the allocator can keep track of things like the size of the chunk allocated and also a constant `magic` value to make sure the chunk was allocated by it. The allocator does some clever pointer arithmetic to move the pointer `sizeof(header_t)` amount before the chunk to reach that metadata.

```c
header_t* hptr = (header_t*)((char*)ptr - sizeof(header_t));
assert(hptr->magic == MAGIC);
```

You can think of the free list as a linked list with a `size` and `next` value. The `size` dictates how much free space there is, and `next` points to the next allocatable chunk. When a `free()` happens, it becomes the new head of the free list and points to the previous head. 

There are several allocation selection algorithms to use:
- Best Fit: Traverse list, find closest space to minimize waste.
- Worst Fit: Find biggest chunk, take whatever you need from that.
- First Fit: First chunk that fits.
- Next Fit: First chunk that fits starting from location where we last were.

**Buddy allocation** thinks of space 2^N. You divide the block by 2 until it fits the desired amount requested with little leftover. When freeing the space, you look to see if the neighboring half is also free, if it is, you coalesce them into one. You do this all the way up the tree.
#### Paging: Introduction 
A **page** is a fixed size unit of address space in virtual memory to simplify memory management. The physical memory side is called a **page frame**.

A **page table** maps between the virtual pages and the physical ones. It's a per-process structure. 

>[!example]
>64-byte address space, meaning there are 64 fillable cubbies. Let's say 16 bytes per page. 
>So a total of 4 pages (64 / 4 = 16)
>Address space is 6 bits long, we need 6 bits to differentiate between 0 -> 64 in binary.
>`000000` -> `111111` (2^6 = 64).
>
>Let's say we had the address `100110`. The first two bits represent which page to look at. In this case, page 3. We will translate page 3 to its physical address. The remaining 4 bits are the offset from that page.
>
>So let's say page 3 translates to 7 page frame. Then the final physical address value would be `1110110`. `111` being 7 (page frame) and `0110` being the offset, this won't change.

The page table has entries (PTE) that give us information about that page. Things like:
- The associated physical page frame (PFN).
- If a frame is valid.
- The w/r/x for the frame.
- If the frame is on disk or in memory.
- If the frame has been modified since last brought to memory.

```
[  VPN  |  PFN  | valid | protec | ASID ]
```

The style of paging introduced so far is too intensive to be utilized, it involves lots of translations using the table and requires lots of memory to store these tables. We will talk about approaches to fix this shortly.

#### Paging: Faster Translations (TLBs)
**Translation Lookaside Buffers** is part of a hardware chips MMU and acts as a cache for frequent translations so that it can be done without consulting the page table.

More architectures will have the operating system handle the cache miss. It'll update the cache and set the program counter back to the same instruction so that it can try again (and not have a cache miss!).

>[!quote]
>_"If the VPN is not found in the TLB (i.e., a TLB miss), the hardware locates the page table in memory (using the page table base register) and looks up the page table entry (PTE) for this page using the VPN as an index. If the page is valid and present in physical memory, the hardware extracts the PFN from the PTE, installs it in the TLB, and retries the instruction, this time generating a TLB hit; so far, so good."_

The **Address Space ID (ASID)** is basically a process identifier so that the TLB doesn't have to flush its entries on every context switch. Otherwise the TLB entries might cause us to acquire memory from the wrong program. Least Recently Used is a common approach for discarding entries.

>[!important] Cache lines vs TLB vs Pages
>A memory page is used to chunk up virtual memory into equal sized blocks. Cache lines are small blocks of memory (usually like 64 bytes) used for quick access by the CPU, stored in the L1, L2, and L3 caches. The TLB is just for translating virtual to physical addresses quickly.
>Memory pages are load into the RAM and stored back to disk by the operating system depending on need. This is not the same thing as evicting an entry in the TLB. The TLB is just removing address translations using a LRU type algorithm.
>Cache lines follow a similar pattern, but all three of these things are **different**.

#### Paging: Smaller Tables
The problem is, page tables take up too much space, we need to find ways to minimize this. 

**Internal fragmentation** is when there's a lot of waste within a page. For instance, if the page size is 16KB and the process only allocates 1KB, then the rest of that 15KB will remain unused.

>[!info] Address Space Terminology 
>32-bit address space means each address has a length of 32. Allowing for 2^32 addresses. 1 address typically stores 1 byte. So an int would fit in a single address.

>[!important]
>If it says "N-bit address space": It's about the length of each address (i.e., how many addresses there can be).
>
>If it says "N-byte address space": It's about the total memory size the system can address.

In a hybrid approach, we have 3 page tables, one for each segment (code, stack, heap). The base and bounds are still used here, but the base acts as a pointer to that segments page table, and the bound is used to dictate *how many valid pages it has*. This is a nice benefit because now, unallocated pages no longer take up space in the page table. 

A **page directory** is a 2-level tree that works by having a top level table that points to / includes chunks of pages with valid allocations. It simply excludes those that are empty. **Page directory index (PDI)** is basically which chunk its in.

The number of PTEs per chunk is page size / PTE size. 

>[!example]
>- 1KB of address space.
>- Page size is 64 bytes.
>- 16 pages total = 1024 / 64.
>- PTE size is 4 bytes.
>- 16 chunk size = 64 (page size) / 4 (PTE size)
>- So the table is divided into 16 chunks each with 16 entries.

A quick example of translating a virtual address to physical address:
- First 4 bits are for the page directory index.
	- 4 bits because there are 16 page chunks.
- Next 4 bits are for the page table entry in that chunk. 
	- 4 bits once again because there are 16 entries to chunk.
	- This will give us the page frame number
- The rest of the 6 bits is the offset value.

```go
vAddr := "11 1111 1000 0000"
vAddr = vAddr.Trim()
chunkIdx := vAddr[:4] // "1111" / 15th chunk
pageTable, ok := chunks[chunkIdx]
if !ok { 
  // page table chunk is invalid! 
}
entry := pageTable[vAddr[4:8]]
pAddr := entry.PFN + vAddr[8:] 
```

#### Beyond Physical Memory: Mechanisms
We want to support the illusion of a large virtual address space. The reality is that the address space of all the concurrently running processes cannot exist in memory, we need to utilize our hard disk.

**Swap space** is a special section on disk that the OS uses to store pages and swap them in/out of main memory. When a page isn't being used, it'll be hot swapped out of memory and into swap space.

>[!important]
>Remember that the virtual address is a facade. The VPN is an index for finding the PTE in the page table. The PTE then helps us get the PFN.  
>
>When a page gets swapped into memory, the page table entry for that page is updated to the correct new PFN. This is nice because the virtual address stays the same, only the table mapping is updated.


The hardware will look at the **present bit** of the PTE to determine if a page is in memory or not. If it's not, this is called a **page fault**. This I/O of moving a page to/from memory is usually a blocking event, so the OS will start running a different process in the meantime.

The **page-fault handler** occurs when the hardware raises a page fault exception, the OS then becomes involved and moves the page to memory. If memory is full, then the replacement algorithm is invoked.



```go
const SHIFT = 
vAddr := "11 0111 0000 0000".Trim()
VPN := (vAddr & VPN_MASK) >> SHIFT
if entry := TLB.get(vAddr); entry != nil {
  entry.
}
```

#### Beyond Physical Memory: Policies
It's important to think of memory as a cache with limited space and having to load from disk as a cache miss. 

The primary eviction algorithms include: LRU, Random and FIFO. LRU is the most optimal. There are many ways it can be implemented.

The **use bit** tells the OS whether or not a page was recently used, 1 = page is referenced, 0 = it has been reset. The **clock algorithm** has pages connected in a cycle. It goes around until it fits a page with the use bit set to 0 and evicts it. 

The **dirty bit** tells us whether a page has been modified in any way. This means there's a diff between this pages contents that exist in memory vs disk. The OS needs to **flush** this page, write the contents back to disk, to make it clean again. 

An eviction of a dirty page is more expensive, because it has to be written back to disk as opposed to the page frame simply being replaced. So the algorithm might look for pages that are not dirty and have been used to evict. 

If a system doesn't have enough space to optimally schedule pages within the confines of the memory size, it's called **thrashing** because it has to constantly swap from disk. 

>[!quote]
>"Some current systems take more a draconian approach to memory overload. For example, some versions of Linux run an out-of-memory killer when memory is oversubscribed; this daemon chooses a memory-intensive process and kills it, thus reducing memory in a none-too-subtle manner."

#### The VAX/VMS Virtual Memory Systems

>[!tip] Seg Faults
>A segmentation fault occurs because a null pointer references address 0. We look that up in the TLB, get a cache miss. We then consult the page table and see that VPN 0 is marked as invalid, thus resulting in the OS terminating the program.

