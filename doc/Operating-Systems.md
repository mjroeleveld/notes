![](./assets/Operating.png)

# Operating systems - three easy pieces course summary

An operating system is a resource manager: it virtualises the CPU and memory for a list of processes. This way, multiple processes can run seemingly (threads on a single core) or actually (multiple cores) in parallel. It also isolates these processes providing security and allocates resources in a way the system along with its programs run well.

## CPU virtualisation
- The OS virtualises resources (memory, CPU, disk) for processes
- A process is an abstraction provided by the OS that can be summarised by taking all the different pieces of the system it affects during its execution
- A process has:
    - A memory space (address space) where necessary (static) code (on disk) is loaded into
    - Registers that instructions read/update
    - An instruction pointer (current instruction), stack pointer
- The process API consists of calls programs can make related to processes, such as creation, destruction, etc
- On process creation, the OS loads a program’s code; creates a stack with the program arguments; allocates heap memory; sets up IO (file descriptors); calls main() function
- A process can be running, blocked (by IO) or ready (not running). The CPU scheduler updates these states when time passes and IO interrupts occur
- The OS maintains a process list with a data structure for each process (process control block), holding information like the register context (last known values); the process state, file descriptors, pointers etc.
- The OS provides an API through system calls. It allows a process to perform a privileged operation. POSIX defines hundreds of these calls, early Unix systems around twenty.
- When a system call is made, the program executes a special trap instruction which raises the privilege level to kernel mode. When finished, the OS calls return-from-trap, returning to user mode.
- At boot time, a trap table is created. It provides the location to system call handlers, exception handlers, interrupt handlers etc. A process specifies a system call number from this trap table when it wants to invoke such a call. This way user code cannot specify a code address to jump to for privileged operations. This way the OS “baby proofs” the CPU by setting the trap handlers, starting the interrupt timer and running processes in restricted mode. Also called Limited Direct Execution
- Processes run using time sharing, they get to run a small portion of CPU time. The OS takes control with a timer interrupt if a system call isn’t made.
- Which processes can be controlled by who is encapsulated in the notion of a user. The superuser (root) can control all processes
- In order for the CPU scheduler to make a context switch to another process, CPU registers, PC counter and stack pointer are saved and restored onto the kernel stack (system call) or process control block (context switch)
- The UNIX API to create processes consists of three system calls:
    - fork(): creates a copy of the current process as a child process (execution resumes from same line)
    - exec(): runs a program different than the current program. It does this by loading code and overwriting the code segment. Heap and stack are reinitialized (no new process is created but current program is transformed).
    - wait(): waits for the child process to complete
- A shell is just a program; it shows a prompt and runs fork() and then exec() for to create a new child process to run the command and then wait() for the command to complete
- Redirection to a file (command > file.txt) in a shell is done by closing stdout and opening a file before exec() happens
- The UNIX pipe() system call (command1 | command2), the output of one process is connected to the input of another process through a queue (pipe)
- Processes can implement signal() to catch various other process signals like SIGINT, or kill(), etc.
- CPU schedulers are often implemented as multi-level feedback queues. Time is divided in slots. All processes start having top priority and are down prioritised as they use a lot of CPU. If a process depends on IO, it will be finished within the bound of its time slot and stay at high priority. High priority jobs get less time allocated than lower priority jobs, which are CPU-intensive jobs. Once in a while all jobs’ priority are boosted to prevent job starvation and to reevaluate the IO/CPU ratio.

## Memory virtualisation (see also CS107 notes)
- A process’ memory is divided into heap, stack and code segments.
- Bases and bounds are the lower and upper limit of a segment's address space. These address ranges are saved in a register and are used/checked during a virtual address lookup. An invalid lookup results in a segment fault.
- Paging divides memory into fixed-sized pages. Processes are unaware of these pages but memory accesses need to be translated to their physical address using page numbers and offsets.
- A per-process page table (e.g. linear array) contains the mapping from virtual page numbers to physical addresses. A page table entry (PTE) may contain flags such as protection bits (read/write), present bit (swapped or not), dirty bit (modified or not) and accessed bit. A memory lookup has first to be translated using this mapping (an address consists out of the virtual page number + offset).
- Page tables are big and slow. With a typical 32-bit address space and 4KB page size, a virtual page number address is 20 bits long.  With an offset of 12 bits that means there are 2^20 translations to remember. With a page table entry of 4 bytes, that is 4MB per process. With 100 processes that’s 400MB.
- A Translation Lookaside Buffer (TLB) is used to cache common page->address translations per process. It’s a small dedicated on-chip cache that is much faster than a lookup in a page table. With this fast cache, memory lookups are almost as fast as if no memory is virtualised at all.
- To preserve memory, multi-level page tables are used. With a multi-level page table, page numbers are chopped up into page-sized units. Then, if all page table entries are invalid, don’t allocate that page of the page table at all.
- The present bit in a PTE is used to indicate whether a page resides in memory or is swapped to disk. If a memory lookup is not present, it results in a page fault and the OS reads the page from disk.
- One swapping policy is LRU (least recently used), where the hardware sets a used bit to indicate that a page is accessed. A background process can then swap unused pages to disk.
- An optimal swapping policy does not exist (LRU, random, all have tradeoffs). Because memory is cheap, the discussion around swapping policies has lessened.
- Linux has kernel virtual memory and kernel logical memory, where logical memory addresses map to physical memory in consecutive order. Virtual memory works with pages and can reside through pages everywhere in physical memory.
- 64-bit Linux has a 4-level page table and supports 4KB, 2MB and 1GB pages. Larger pages are nowadays often needed for big memory workloads to increase TLB coverage.
- Linux uses a page cache to keep pages in memory from three sources: memory-mapped files, metadata from files/devices and heap and stack pages from processes. It keeps these popular items in memory to reduce costs of accessing persistent storage. A background process flushes dirty cached pages back to storage (file or swap).
- Linux uses a 2Q system where an active list and an inactive list of pages are maintained. Only items from the inactive list are swapped which ensures no active pages are swapped which could happen with a regular LRU policy. E.g. when a big file is repeatedly accessed, LRU will kick out all other active pages that may be more relevant.
- Linux randomises the order of the stack, heap and code segments in a process’ virtualised memory to prevent security attacks like buffer overflows.

## Concurrency (see CS107 notes)
- Hardware provide instructions to atomically set/read values, allowing locks to be implemented.
- Locked data structures can be implemented using a single lock for the whole structure or otherwise more fine-grained (per bucket for a hash table).

## Persistence and I/O
- Perhipherals are connected to the CPU through buses. Memory through a proprietary memory bus. High performance devices such as GPU through a generic IO bus such as PCI. Peripherals through a peripheral IO bus such as SATA, USB.
- An IO device exposes its API through registers and abstracts its internal structure such as a microcontroller, memory etc. this way
- A device may have a status register for polling, but interrupts change this interaction by raising a hardware interrupt causing the CPU to jump to an interrupt handler
- Communication with IO devices happens through either special IO instructions or through memory mapped IO, where device registers are mapped to memory addresses.
- An OS implements device specifics through device drivers. Drivers make up for a big part of kernel code, approximated at over 70% of Linux kernel.
- IO communication in the OS happens out of several layers of abstraction; an application doing a system call (POSIX) is completely oblivious to the specifics of which disk class it is using; it simply issues block read and write requests to the generic block layer, which routes them to the appropriate device driver
- Hard Disk Drives (HDDs) consist out of a large number of sectors (512-byte blocks), each of which can be read or written. They are numbered from 0 to n, which forms the address space of the drive.
- HDDs have rotational delay and seek time. A disk surface consists out of tracks laid out in circles, with hundreds of tracks in the width of a human hair. Seek time is the time the rotational head needs to move to another track. On HHD’s, sequential workloads are way faster than random workloads.
- The OS schedules disk requests and decides which requests to schedule next. The OS can estimate how long a disk request will take and therefore tries to schedule the shortest request first.
- A file descriptor is an integer returned by open(), private per process, used in Unix to access files. I.e. a pointer to an object of type file, which you can use in write() and read().
- Each process has an open file table with a struct for each open file with info such as r/w flags and offset. read() and write() update an offset value in this table, serving as a cursor for the current process.
- A file is an array of bytes which can be created, read, written and deleted. Each file has an inode with an inode number associated with it.
- A file has access bits, defining access for the owner, group and everyone. These can be modified with chmod by the owner or superuser. More advanced filesystems like AFS use access control lists for more granular access control.
- An inode is a persistent data structure (file system dependent) that holds metadata about a file. E.g. device id, user id, group id, file size, last access t time, etc.
- A directory contains a list of user-readable, low-level name (inode number) pairs. A directory always has the special entries `.` (self) and `..` (parent)
- File systems often treat directories as special type of files, with an inode with its type specified as directory
- When creating a file, it will create an inode for that file, and then links a human-readable name to that file and put that in a directory
- When creating a hard link using ln, a new name in a directory is created with the same inode number of the original file.
- When deleting a file, unlink() is called which removes the linked name from the directory. The inode contains a reference count; only when the count reaches zero, the inode and other data structures of the file are deleted.
- By the OS, symbolic links are of a special type (other than dirs and files) and are defined by just the path linked to. Unlike hard links, symbolic links can link to directories
- Using makefs, an empty file system can be created at e.g. a partition. Then, using mount, this file system can be mounted at a path and is unified with all other file systems
- Regular file systems are partitioned into blocks. It contains a superblock containing filesystem metadata, inode blocks (where inodes are stored in), data blocks and blocks with an allocation structure that hold information about which inode/data blocks are allocated (bitmaps).
- Within inodes, a data structure is used to store where a file’s data blocks are located. Many file systems use a multi-level index consisting out of pointers (to pointers) to all the file’s data blocks. Other file systems store data blocks contiguously and store just the starting address plus length.
- Upon open(), a file descriptor is created. Traversal starts at the root which has a fixed inode number. The filesystem recursively traverses inodes (directories) and reads the last inode into memory. A subsequent read() will consult the inode to find the location of the data block at offset 0, update last accessed time, etc. Upon close(), the file descriptor will be deallocated
- Popular file system blocks are cached into memory alongside virtual memory pages in a unified page cache.
- Several ways to make operations faster are used, such as grouping data on tracks close in close proximity (cilinder groups). This way, many disk seeks can be largely prevented for sequential reads.
- Many file systems use journaling as a solution to recover after a crash. Transactions are written to a log (journal) which can be replayed if a crash occurs. A transaction id, the inode, bitmaps and optionally the data block are written to the journal between a start and end block. Once committed, the pending changes are written to disk.
- Log based File Systems work by writing inodes and data to disk in sequence, as in a log. In contrast to regular file systems, data blocks are not overwritten but buffered in memory and written contiguously to an unused portion on disk. This enables high performance for writes.
- LFSs work by adding a inode map to every written segment. At a fixed location on disk, pointers to all inode maps live from which the entire inode map can be constructed.
- LFSs generate garbage; old copies of data are scattered around the disk. A garbage collector compresses segments with few used data blocks and frees them.
- LFSs advantages:
    - Fast write performance (nowadays many blocks are cached in memory and thus most operations are often writes)
    - Faster for random writes (seek and rotational delay are expensive)
    - Faster for common workloads (e.g. a large number of writes in regular file systems result in many operations to blocks all over the disk, limiting performance)
    - Works better with RAID and SSDs
- SSDs store data on flash chips. SLC flash stores a single bit in a transistor, MLC two bits, TLC three bits.
- SSDs have blocks (e.g. 128KB) containing pages (e.g. 4KB). To write a page, a whole block has to be erased.
- Flash chips wear out as the transistor charge changes over time (MLC and TLC wear faster). That’s why writes have to be spread evenly over the chips to level wear.
- Most SSDs are log structured. Writes are done sequentially to erased blocks. Also, some often read blocks are migrated to ensure all blocks are evenly written to.
- For reads the SSD keeps a table with mapping table with logical to physical page/block numbers.
- Garbage collection cleans up duplicate (old) pages.
- SSDs are faster for sequential IO and much faster for random IO. However, more expensive.
- IO devices adopt checksumming to detect data corruption. RAID can be used to recover from data loss.
- Network File System (NFS) is a file system that adheres to the POSIX file system interface (open(), read(), write(), etc.). It is a stateless and idempotent protocol. Therefore servers can quickly restart and clients can simply retry requests. 
