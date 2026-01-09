---
hasThumbnail: false

---

## Overview

PintOS is an instructional operating system used in UC Berkeley's CS 162 class to help lear how real kernels support user programs, system calls, processes/threads, scheduling, and synchronization. In my Operating Systems coursework, I worked with Dhruv Agarwal, Anshul Jambula, and Nemerjit Singh to implement the major functionality across the kernel/user boundary, including:

- User program support (argument passing, process lifecycle)
- A robust system call interface (process control + file operations)
- Safe user-memory access patterns so invalid pointers can’t crash the kernel
- Floating-point (FPU) context management across traps/context switches
- Threading/scheduling improvements (efficient sleep, strict priority scheduling, priority donation)
- Additional tests beyond the starter suite

## Project Breakdown

### 1) User Program Support

The first step was to make user programs receive arguments exactly like a normal process.
- command-line parsing into `argc/argv`
- building the initial user stack layout so `main(argc, argv)` works as expected
- consistent startup and exit behavior so processes can terminate cleanly and report status  

### 2) System calls

Next, we built out the syscall path so user programs could ask the kernel for services—process control and I/O.

Syscall support was added for:
- basic process control (terminate, start a new program, wait for completion)  
- file operations (open/close/read/write, etc.)  

A challenge here was that syscalls are a trust boundary. User code can pass garbage pointers or malformed arguments, so the kernel has to make sure the inputs are valid, and we added user-memory validation to prevent kernel crashes, such as:
- rejecting null/unmapped pointers and kernel addresses
- handling edge cases like partially-valid multi-byte reads across page boundaries
- terminating the offending user program rather than panicking the kernel  

### 3) Process coordination and correctness guarantees

Process creation and waiting sematnics require careful coordination internally. In particular they require the parent and child to agree on what happened and when.

We implemented conceptual guarantees like:
- the parent can reliably learn whether the child successfully started (load succeeded vs. failed)  
- waiting returns the child’s exit status, but only for direct children and only once per child
- resources are reclaimed correctly even if the parent never waits (to avoid leaks)  


### 4) File I/O model

After process control, we implemented a usable file I/O interface. The goal was to give each process a file descriptor table model and support both console I/O and file-backed I/O.

Features added:
- a full descriptor-based interface: create/remove/open/close, read/write, seek/tell/filesize  
- standard input/output behavior (keyboard input + console output)  
- per-process descriptor bookkeeping so different opens have independent file positions and descriptors don’t implicitly carry across processes  

At this point, the underlying filesytem layer wasn't thread-safe, so we ensured that file syscalls were synchronized to prevent races/corruption. We also made sure to enforce the common OS behavior that prevents modifying an executable while it is running. 

### 5) Floating-point support

PintOS starts out focused on integer CPU state. We added FPU support so that floating-point code can run correctly even with interrupts, syscalls, and thread preemption.

This required:
- initializing the FPU state at startup
- ensuring each thread has its own floating-point register state
- saving/restoring that state on context switches and when entering/exiting the kernel  

This validated by using a floating-point computation task (approximating *e*) to confirm preemption doesn’t corrupt floating-point state. 

### 6) Threading and scheduling improvements

After this initial setup for enabling the system to run programs, our next set of tasks was making it work well under load and contention.

#### Efficient sleeping (alarm clock)

This replaced inefficient sleeping (looping/yielding while waiting) with real sleeping where a blocked thread consumes no CPU until it’s time to wake up. The wakeups are driven by timer ticks.  

#### Strict priority scheduling + priority donation

We implemented strict priority scheduling so the highest-priority runnable thread runs first, with round-robin among threads of equal priority. 

To make synchronization behave correctly under priorities, we ensured:
- synchronization primitives unblock the highest-priority waiter first 
- priority donation prevents priority inversion when a high-priority thread is blocked on a lock held by a lower-priority thread, including nested donation and multiple donors, with proper donation removal on unlock 
- threads yield when they no longer have the highest effective priority 

### 7) User-level threads + user synchronization (pthread-like model)

To support multithreaded user programs, we added user threads with a 1:1 mapping to kernel threads (so each user thread is scheduled by the kernel). 

That included:
- creating threads within a process
- joining on thread completion (including joining threads that already exited)
- defining exit semantics, including what happens when the main thread exits  

We also added user-accessible locks and semaphores through syscalls. Each operation can cross into the kernel, but the behavior is correct and usable for coordination in user programs. Since each user thread needs its own stack, we also implemented the conceptual memory strategy for placing multiple stacks within a process’s address space without collisions. 

### 8) File system internals

In the earlier portions of the project, we mostly used the file system through syscalls. In the third phase, we went a step further and implement core file system behavior such as caching, file growth, and directory trees. 

#### Buffer cache

By default, many file operations would hit the disk device repeatedly. We added a **buffer cache** so the OS can keep recently-used disk blocks in memory and avoid redundant disk I/O.

What we built:
- a cache of **up to 64 disk blocks** (fixed capacity)
- a **write-back** policy (writes accumulate in memory and are flushed on eviction), instead of write-through
- a replacement policy based on locality (e.g., clock/LRU-style behavior), rather than FIFO/random
- full integration so disk accesses go through the cache (not just a couple of call sites)

A big part of this work was making it correct under concurrency:
- if one thread is reading/writing a cached block, other threads shouldn’t evict or race that entry
- if a block is currently being loaded, other threads shouldn’t load duplicates or read a half-filled cache entry

We also removed extra “bounce buffer” copying so reads/writes can operate cleanly through the cache and the disk interface.

#### Extensible files

The starter file system allocates files as one contiguous extent, which prevents file growth. We modified the on-disk representation so files can grow over time without requiring a single contiguous region of free blocks. 

This enabled:
- files can start at size 0 and **grow when a write goes past EOF**, like a normal OS
- support for **fast random access** (so designs like FAT are discouraged in the spec)
- support for **large files up to ~8 MiB**, which pushes you toward an inode that can reference data through multiple levels (direct/indirect/doubly-indirect style indexing)
- correct semantics when writing past EOF: the file extends, and any gap is filled with zeros
- robust handling of “out of memory” and “out of disk space” situations by aborting/rolling back safely, without corrupting metadata or leaking blocks

This work also removed a limitation of the basic root directory implementation (which starts out with a small fixed capacity), because directories are files too and now can expand as needed.

#### Subdirectories + path resolution (hierarchical directories)

The starter system technically has directories, but user programs can’t really use them. We added real support for a hierarchical directory tree and made the OS understand paths similar to what you would expect from a Unix-like system.

Features added:
- **hierarchical directories** (directories can contain files and other directories)
- **absolute and relative paths** for syscalls that take path arguments
- support for special path components **`.` and `..`**
- a **per-process current working directory**, inherited by child processes on exec, then independent afterwards

On the interface side, we added directory-focused syscalls so programs can navigate and inspect directories (change directory, make directory, read directory entries, check whether an FD is a directory, and retrieve inode numbers).

We also updated existing behaviors to make directories first-class:
- opening a directory should be allowed, but reading/writing file bytes through that FD should not (directory operations use directory-specific syscalls instead)
- removing a directory should work for **empty directories**

#### Synchronization (removing the “global lock” approach)

As the file system became more complex (buffer cache + growing inodes + directories), correctness under concurrency mattered more. Instead of relying on a single global lock, we switched to a more careful synchronization approach across the file system components.

## Testing and validation strategy

We used the provided test suite as the baseline and added additional tests for behaviors such as:
- invalid user pointers and page-boundary edge cases 
- coordination rules for starting/waiting on processes 
- filesystem concurrency constraints 
- priority inversion/donation corner cases 
- floating-point correctness across preemption 
