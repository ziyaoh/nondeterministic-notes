# Operating System: Three Easy Pieces

## CPU Virtualization

### Process

- Components
    - memory (address space: the memory that the process can address): instructions, data
        1. stack
        2. heap
    - registers
    - I/O stuff (list of files, inodes)

- Process State (roughly): Running, Ready, Blocked

- Process Object in OS: for process current state
    - register content: for context switching
    - process state
    - open files and inodes
    - etc.

- API
    * fork(): creates exact same copy of the the parent process, both running to the point of fork(), with parent returning PID and child returning 0
    * wait() / waited(): block until the waitee exits
    * exec(): reinitialize current process to another one, without create one new
    * kill()

    Motivation to seperate fork() and exec(): empowers Unix shell of many features by allowing it run code between these two calls

### Limited Direct Execution
Problem: how to do time sharing **efficiently** with **control**? i.e. protect sensitive resources from user program & stop it from running forever

#### Direct Execution
run user program on CPU, efficient

#### Restrict Operation
- two processor modes: user mode and kernel mode
- user program runs in user mode, OS runs in kernel mode
- user mode is restricted in permissible operation, has to go through kernel by making **system call**

user mode -> syscall -> trap -> kernel mode -> operation -> return from trap -> user mode

*How does trap know what to execute when called?*

Kernel set up a **trap table** in kernel mode at boot time, telling hardware what code to run when certail exceptional event occur (**trap handler**).

#### Time Sharing for Real
**Time interrupt** and **interrupt handler** to stop a user program from running forever. OS start a timer to send interrupt signal periodically at boot time, which will be trapped to the interrupt handler.

#### Context Switch
Save registers properly to restore current process on CPU safely when switching to a different process.

On x86:
user mode -> trap -> save registers on kernel stack ->
- if interrupt handler -> save register to Proc_t obj -> switch context by changing the stack pointer to a different process's kernel stack ->
- if other handler -> do the required operation ->

-> return from trap -> user mode

Concurrency issue: refer to the second part of the book. (*TODO*)

### Scheduling
*TODO*

## Memory Virtualization

Memory virtualization means an illusion to the user program that it has access to a large contiguous address space to put its code and data into. This makes the user program easier to write, and provide isolation and protection.

### Address Space

The address space of a process contains all of the memory state of the running program: 
- code
    
    instructions

    static, placed at the begining of the address space

- stack

    current state, local variables etc.

    start at the end of the address space, growing backwards

- heap

    dynamically-allocated, user-managed memory

    start after code portion, growing forwards

More to consider when threads gets involved.

#### Memory API

Stack memory is managed by compiler in C. Heap could be allocated and freed with the following function calls.

- malloc() and free()

    library calls built on top of some system calls (brk() for example, which changes the location of the program's break: the location of th end of the heap)

- mmap()

    create an anonymous memory region that's associated with swap space (*todo*)

- calloc() and realloc()

### Address Translation

**Virtual address** is the location of data in memory from program's point of view. **Physical address** is its actual location in real memory.

Address translation transforms each memory access by changing the virtual address to physical address.

#### Base and Bound
Use a pair of base and bound registers for each process to do address translation. Base is the start of the address space in physical memory, bound is the end.

`physical address = virtual address + base`

Assumes that address spaces are allocated continguously in physical memory, and that size of address spaces are smaller than that of physical memory.

Problem: leads to internal fragmentation when stack and heap is small, wasting lots of spaces.

#### Segmentation
A pair of base and bound registers for each portion in the address space, together with some other properties to prevent interval fragmentation.

| Segment | Base | Size (Bound) | Grows Positive? | Protection |
| --- | --- | --- | --- | --- |
| Code | 32K | 2K | 1 | Read-Execute |
| Heap | 34K | 2K | 1 | Read-Write |
| Stack | 28K | 2K | 0 | Read-Write |

Problem: leads to externel fragmentation due to variable size of each segments.

### Paging
Paging should be another address translation strategy I guess but important enough to be mentioned separately.

Paging view physical memory as an array of fixed-sized slots called page frames, each of which can contain a single virtual-memory page.

#### Page Table
A per-process data structure to translate virtual address to physical address

To translate a virtual address, the address is splitted into two components: the **virtual page number (VPN)**, and the **offset** within that page. Page table translates VPN to **physical frame number (PFN)**. Append PFN with offset gives us the corresponding physicall address.

#### Linear Page Table
An array of **page-table entries (PTE)**, with their indices being their VPN.

PTE
- valid bit: whether this page is actively being used
- protection bit: whether the page could be read fromt, written to, or executed from
- present bit: whether this page is in physical memory or on disk
- dirty bit: if the page has been modified since it was brought into memory
- reference bit: track if a page has been accessed, useful in determining which pages are popular and should be kept in memory

Problem: This kind of page table often takes too much memory especially as a per-process data structure. It also slow thing done because to access data in memory, we have to go to the page table implicitly first to do the translation, which slow memory access by factor of two or more.

#### Translation-lookaside Buffer (TLB): faster translation
A cache of popular virtual-to-physical address translations in the chip. Making the address translation faster on average.

When an address translation is required, looks VPN up in the TLB first. If TLB hit then done. Otherwise, go to the page table and bring the PTE into TLB (if the memory access is legal), then retry the cache look up.

A good TLB hit rate relies on **spatial locality** and **temporal locality**.

##### TLB During Context Switch

To prevent one process to access memory belonging to other processes, TLB need to be flushed when we switch context, by setting all valid bits to 0. This operation may cause more TLB misses though. Therefore, some system supports sharing of the TLB across context switches, with the **address space identifier (ASID)** field inside.

##### Replacement Policy
- least recently used (LRU)
- random
- etc.

#### Smaller Tables
- bigger pages
    
    Less PTE, but more internal fragmentation

- Hybrid: Paging and Segments

    Instead of a single page table for the whole address space, assign one page table to each logical component: stack, heap, code.. 

    This approach saves space when the process only utilize a compact small portion of its address space. External fragmentation could arise again, since page table is now of variable size.

- Multi-level Page Tables

    Chop up the page table into page-sized units; if an entire page of page-table entries in invalid, don't allocaate that page of the page table at all. Use a new **page directory** structure to track the pages containing PTEs.

    This approach saves spaces when process uses a small portion of its address space. It also avoid the external fragmentation problem above. But in case of TLB miss, two loads from memory is required to do the translation, instead of just one load with the simple linear page table.

- Inverted Page Tables

    A single page table that has an entry for each *physical page* of the system. The entry tells us which process is using this page, and which virtual page of that process maps to this physical page.

    A hash table is often employed to spped lookups, since scanning through the page table would be expensive.

### Free Space Management
Free list is a list of free blocks on memory available upon a call to malloc().

A free block could be splitted if user requests a small chunk of memory. When user frees a block of memory, coalescing need to be performed, meaning that the newly free block should be merged with adjacent free block currently on the free list if possible.

A malloced block is preceded by a header, containing the size of the block and some other info. When free() is called, compiler looks for the header and knows the size of the block need to be freed. Similarly if user calls `malloc(size)`, the actual consumption of block is `size + sizeof(header)`.

Free list node are represented in the free space itself.

Basic malloc strategies
- best fit / smallest fit

    use a block that's both greater or equal to and closest to what the user asks

- worst fit

    use the largest chunk

- first fit

    use the first chunk that's big enough, always start from the begining of list

- next fit

    similar to first fit but use an extra pointer to indicate where the next malloc search should start from

Other approaches

- Segregated Lists

    if a particular application has one (or a few) popular-sized request that it makes, keeps a separate list to manage object of that size; all other requests are forwarded to a more general memory allocator

    eg. slab allocator used in Solaris (and Weenix), allocating locks, file-system inodes, etc.

- Buddy Allocation

    divides tha free space up into chunks of size 2<sup>n</sup>, and return the smallest chunk that's big enough

    makes coalescing much easier, but suffers from internal fragmentation

### Swap

The addition of swap space allows the OS to support the illusion of a large virtual memory for multiple concurrently running processes.

In case of a memory access, if we have a TLB miss and the PTE in page table shows the requested page is not present in physical memory (by the present bit), a page fault occurs and triggers the page fault handler. In this case, the PTE should have the disk address of the page in disk. OS could issue a disk I/O to fetch the page and swap it into memory (**page in**), update the page table, and retry the instruction.

#### Page-Replacement Policy
To perform a page in, if the memory is full, OS needs to first **page out** some pages to make room. This process is usually performed by a background thread asynchronously.

This policy is critical because disk I/O is way slower than memory access, and we want to minimize the number of page in operation as much as possible. The theoritical optimal replacement policy is to replace the page that will be accessed furthest in the future. This policy leads to the fewest number of misses (and fewest number of page in) overall. This approach is just theoretical though since we can't know the future.

More practical policies
- FIFO
- Random
- LRU

    Precise LRU is too expensive to perform. Systems usually approximate LRU with the help of the **use bit**. e.g. **clock algorithm**

    Some system also considers dirty pages during swaping. Unmodified pages are easier to page out because we don't need to write it back to disk.

#### Other VM Policies

- Page Selection Policy

    the policy to decide when to bring a page into memory

    - demand paging
    - prefetching

- how the OS write pages out ot dick
    - one page at a time
    - clustering writes

        collect several pending writes and write them all at once

### The VAX/VMS Virtual Memory System (Case Study)
*TODO*

## Concurrency

### Thread

Threads within the same process share the same address space. Each of them has a separate stack in the address space.

Context switch between threads is pretty similar to that between processes. One difference is that we don't necessarily switch page table for threads.

API
- Create
- Join
- Locks
    - Initialize
    - Lock
    - Unlock
    - Destroy
- Conditional Variables

    Useful when some kind of signaling must take place between threads. A lock is required together with a cond var.
    - Wait
    - Signal
    - Broadcast

### Locks
- spin lock
- mutex
- futex
- two phase lock

    try spin-wait first then sleep

### Thread Safe Data Structures

- Scalable Counting

    Simply add a mutex to a counter is not scalable especially in multi-core machines. A **Sloppy Counter** employs a local counter for each core, a global counter, and a lock for each counter. Increment/Decrement operations changes the local counter, which get updated to the global counter periodically. This updating period incurs trade-off between performance and accuracy, therehen the name Sloppy Counter.

- Concurrent Linked List
- Concurrent Queue
- Concurrent Hash Table

    Concurrent Hash Table could be built on top of concurrent list, with a lock on each hash bucket.

### Conditional Variables
Always use while loop to check condition with condition variable.

### Semaphores
A semaphore is an object with an integer value that we can manipulate with two routines. It should be initialized to some value, which determines its behavior.
- sem_wait()

    decrement the int value, and suspend caller if the value is negative

- sem_post()

    increment the int value, and wake a waiting thread up if there is one

Usage
- Binary Semaphores (Locks)

    semaphore initialized to 1

- Semaphores as Condition Variables

    initial value depends, really confusing to me, I would try avoid semaphore personally...

### Reader-Writer Locks
More complication and overhead than simple mutex. Perhaps doesn't give much performance improvement.

### Common Concurrentcy Problems
- atomicity violation
- order violation
- deadlock

## Persistence

### I/O Devices
Components in a computer system are connected via a hierarchical structure: multiple buses, which are communication system that transfer data between components inside a computer.
- memory bus

    CPU, memory

- I/O bus

    graphics and higher-performance I/O devices

- peripheral bus

    SCSI, SATA, USB

#### A Canonical Device
- hardware interface: registers that allows OS to control the device

    - Status: current status of the device
    - Command: tell the device to perform a certian task
    - Data: to pass data to, or get data from the device

- internal structure

#### Communication Protocol
- programmed I/O (PIO): synchronous communication with device

    busy waiting for device ready to receive a new command, issues the command, then busy waiting until it's done

    easy but inefficient on slow devices

- interrupt: async communication

    OS issues a request then sleep the calling thread, and switches to another thread. Device raises a hardware interrupt after it's done. An interrupt handler handles this, finish the request and wake the calling thread.

    Good for slow devices but creates overhead with fast devices, with the extra context switching, interrupt, etc.

    One optimization is to coalesce multiple interrupts into a single one.

- hybrid

    PIO for a little while and then use interrupt

**Direct Memory Access (DMA)** engine can orchestrate data transfers between devices and main memory without much CPU intervention. CPU can thus do some other work while we save data from memory to other device (say disk).

#### Communication between OS and Devices
- Explicit I/O Instructions

- Memory-Mapped I/O

    hardware maps device registers to memory locations, so the os just read or write to the memory address

**Device driver** abstract away the detail of different device interfaces. Filesystem (and applications above) simply issues block read/write requests to the generic block layer, which routes them to the appropriate device driver, which handles the details of issuing the specific request.

### Hard Disk Drives

#### Basic Geometry
- platter
- surface
- track
- disk head & disk arm

I/O time inside disk: `seek + rotational delay + transfer`

**Cache** or **track buffer** is some small amount of memory which the drive can use to hold data read from or written to the disk.

Cahce for writes: **write back** vs **write through**

#### Disk Scheduling
OS orders the I/Os issued to the disk to minimize latency, according to a greedy principle of **SJF (shortest job first)**. Mostly in old OS I guess...

- SSTF: Shortest Seek Time First (nearest-block-first)

    issues close-by I/O requests first

    cause a problem of starvation

- Elevator (SCAN)

    moves back and forth across the disk servicing requests in the order across the tracks

- SPTF: Shortest Positioning Time First

    consider both seek time and rotational delay, usually performed inside a drive

### Redundant Arrays of Inexpensive Disks (RAIDs)
Use multiple disks in concert to build a **faster**, **bigger**, and **more reliable** disk system.

*TODO*

### Storage Virtualization
#### Filesystem Syscalls

#### Disk Structure
- Super Block
- Inodes
- Data Blocks

#### Access by Path
- Read

    1. open: traverse through the path to the target inode, copy inode into memory
    3. read: consult the inode for location of data block, read from disk, manage offset
    4. close

- Write

    even more disk I/Os required

#### Caching and Buffering
Modern systems employ a dynamic partitioning strategy in the memory as a cache for disk blocks for reading purpose.

Write buffering is applied to optimize disk write I/O, by delay and batch disk write requests. One trade-off to consider: durability vs performance. Longer delay means better performance, but higher data loss risk in case of system crash while delaying.

### Fast File System
Optimization of the internal filesystem structure and allocation policies to be "disk aware", but still adhere to the existing interface.

**cylinder group & block group**

consecutive read within the same block group doesn't take long time seeking.

**Policy**

keep related stuff together, and unrelated stuff far apart

**Problem**
Internal fragmentation could be a problem, and **sub-blocks** comes to rescue.

**Parameterization**
interleave data blocks with free blocks to optimize for consecutive reading

### Crash Consistency and Recovery
**Journaling** write down a little note describing what you are about to do somewhere on disk, before overwriting the structures in place.

#### Data Journaling

1. Journal write

    write the transaction, including a transaction-begin block, all pending data and metadata updates, to the log; wait for these writes to complete

2. Journal commit

    write thhe transaction commit block (transaction-end block) to the log; wait for write to complete

3. Checkpoint

    write the pending metadata and data updates to their final locations in the file system

4. Free

    mark the transaction free int he journal by updating the journal superblock

Journal committing is applied to handle system crashes while writing the journal.

#### Metadata Journaling

During the journal write step, only log pending metadata updates, **wihtout** data block updates, thus preventing double writes for each data block. Data writes happen before journal writes

1. Data write
2. Jounal metadata write (without data block updates)
3. Jounal commit
4. Metadata checkpoint
5. Free

### Log Based Filesystem
*TODO*

### Data Integrity and Protection
*TODO*

## Distribution
### RPC

### Distributed Filesystem

#### Sun Network File System (NFSv2)

NFSv2 file server is stateless, leaving client filesystem to do the state management. This is to help with simple server crash recovery.