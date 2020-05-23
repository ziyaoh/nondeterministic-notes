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

Concurrency issue: refer to the second part of the book. (TODO)

### Scheduling
TODO

## Memory Virtualization

