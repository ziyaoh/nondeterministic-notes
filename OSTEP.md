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

#### Paging
*TODO*

#### Free Space Management
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