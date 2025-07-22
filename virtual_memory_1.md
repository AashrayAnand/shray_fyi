# Refreshing Virtual Memory Concepts

I wanted to quickly write up some notes about virtual memory at a high level as I was looking into how the linux ```mprotect``` interface works, for modifying the memory access protection level for specific areas of a process's virtual memory.

This originally came up when I was looking into the write up [here](https://stackoverflow.com/questions/18986264/mprotect-on-a-mmap-ed-shared-memory-segment), which does a good job of explaining why ```mprotect``` will not affect shared-memory access for all processes that memory map a shared memory segment, and instead only will apply on the process which applies the ```mprotect```.

Ulrich Drepper's ["What Every Programmer Should Know About Memory"](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) is a dense, but invaluable reference to deeply understand what I briefly describe here. 

## Introduction to Virtual Memory

Virtual memory is a memory management technique that provides an abstraction of the storage resources available to a program. It maps memory addresses used by a program, called virtual addresses, into physical addresses in computer memory.

Key benefits of virtual memory include:
- Process isolation
- Efficient memory utilization
- Ability to use more memory than physically available through paging
- Memory protection between processes
- Simplified memory allocation

## Virtual Memory Mapping in Linux

Linux uses several key structures to manage virtual memory areas. The most important ones are `vm_area_struct` and `mm_struct`.

### The vm_area_struct

This structure represents a contiguous range of virtual memory:

```c
struct vm_area_struct {
    struct mm_struct *vm_mm;    /* Associated mm_struct */
    unsigned long vm_start;     /* Start address */
    unsigned long vm_end;       /* End address */
    unsigned long vm_flags;     /* Flags */
    struct file *vm_file;       /* Associated file (if any) */
    void *vm_private_data;      /* Private data */
    struct vm_operations_struct *vm_ops;
    /* ... other fields ... */
};
```

### The mm_struct

This structure represents the entire memory management context for a process:

```c
struct mm_struct {
    struct vm_area_struct *mmap;        /* list of memory areas */
    struct rb_root mm_rb;               /* red-black tree of VMAs */
    struct vm_area_struct *mmap_cache;  /* last used memory area */
    unsigned long free_area_cache;      /* 1st address space hole */
    pgd_t *pgd;                        /* page global directory */
    atomic_t mm_users;                 /* How many users? */
    atomic_t mm_count;                 /* How many references? */
    int map_count;                     /* number of VMAs */
    struct rw_semaphore mmap_sem;      /* mmap semaphore */
    unsigned long start_code;          /* start address of code */
    unsigned long end_code;            /* final address of code */
    unsigned long start_data;          /* start address of data */
    unsigned long end_data;            /* final address of data */
    unsigned long start_brk;           /* start address of heap */
    unsigned long brk;                 /* final address of heap */
    unsigned long start_stack;         /* start address of stack */
    /* ... many other fields ... */
};
```

## TLB Cache

The Translation Lookaside Buffer (TLB) is essentially a small, fast cache that stores recently used virtual-to-physical address translations. Modern processors often implement a multi-level TLB hierarchy with separate L1 TLBs for instructions and data.

```
L1 TLB (Split)
Instruction TLB (iTLB)        Data TLB (dTLB)
+-------------------+        +-------------------+
|  Virtual  | Phys  |        |  Virtual  | Phys  |
| Page Addr | Frame |        | Page Addr | Frame |
+-----------------+-+        +-----------------+-+
| VPN1 | PPN1 | F |         | VPN1 | PPN1 | F |
| VPN2 | PPN2 | F |         | VPN2 | PPN2 | F |
| ...  | ...  | F |         | ...  | ...  | F |
+-------------------+        +-------------------+
        |                            |
        v                            v
        Unified L2 TLB (Larger, slower)
        +--------------------------------+
        |  Virtual  | Physical  |        |
        | Page Addr |  Frame    | Flags  |
        +--------------------------------+
        | VPN1 | PPN1 | ASID | R/W/X/G  |
        | VPN2 | PPN2 | ASID | R/W/X/G  |
        | ...  | ...  | ...  | ...      |
        +--------------------------------+

TLB Entry Detail:
+------------+------------+------+-----+-----+-----+-----+
| Virtual PN | Physical  | ASID | R  | W  | X  | G  |
|           | Frame Num  |      |    |    |    |    |
+------------+------------+------+-----+-----+-----+-----+
    20-40        40         16    1    1    1    1   bits

Flags:
R: Readable
W: Writable
X: Executable
G: Global page
ASID: Address Space ID (process identifier)
```

The TLB is fully associative, meaning any virtual page number can map to any TLB entry. When a virtual address is accessed:
1. The virtual page number is extracted
2. All TLB entries are checked in parallel
3. If found (TLB hit), the physical frame is used
4. If not found (TLB miss), the page table is walked

Note: Actual TLB sizes and structure vary by processor. For example:
- A typical L1 dTLB might have 64 entries
- L1 iTLB might have 128 entries
- L2 TLB might have 1024-1536 entries

## Page Tables

The page table is a hierarchical data structure that maps virtual addresses to physical addresses. On modern x86_64 systems, this typically involves a 4-level page table, where each level is an array of 512 entries (2⁹ entries, since each level uses 9 bits as an index).

Each level's entries contain either:
- A physical address pointing to the next level's table (for PGD, PUD, PMD)
- The actual physical page address and metadata (for PTE level)

```
Virtual Address (48 bits)
+--------+--------+--------+--------+------------+
|   PGD  |  PUD  |  PMD  |  PTE  |   Offset   |
|  9bits | 9bits | 9bits | 9bits |   12bits   |
+--------+--------+--------+--------+------------+
    |        |        |        |         |
    v        v        v        v         v
+------+  +------+  +------+  +------+  
| PGD  |->| PUD  |->| PMD  |->| PTE  |-> Physical Page + Offset
+------+  +------+  +------+  +------+  
CR3      

Each table is an array of 512 entries:
PGD[0..511] -> each entry points to a PUD
PUD[0..511] -> each entry points to a PMD
PMD[0..511] -> each entry points to a PTE
PTE[0..511] -> each entry points to a physical page

Example entry (x86_64):
+--------------------------------+----+
|     Physical Page Address      |Flags|
|          40 bits              |12bit|
+--------------------------------+----+
```

During address translation, each 9-bit segment of the virtual address is used as an index into its corresponding table. For example:
- Bits 39-47 index into the PGD array
- Bits 30-38 index into the PUD array
- Bits 21-29 index into the PMD array
- Bits 12-20 index into the PTE array
- Bits 0-11 provide the offset into the final physical page

This hierarchical structure allows for efficient memory usage, as tables for unused portions of the address space don't need to be allocated.

## Virtual Memory Address Translation

The process of translating addresses in a process's virtual memory address space, to the corresponding physical memory address is roughly as follows:

1. CPU checks if the virtual address has a TLB cache entry.
2. CPU checks if the virtual address has a page table entry.
3. If the TLB cache and page table did not have entries for the virtual address, invoke the ```page_fault_handler```, to consult the process's virtual address space information further on how to address the access, looking up the corresponding virutal memory area for the address:
    - If the address's physical page is "swapped out" (written to disk), allocate a page in physical memory, read in the page, and update the page table.
    - If the address does not correspond to any virtual memory area, a segmentation fault occurs.
4. Check whether the caller's access type matches the permissions for the page, otherwise a fault is generated. 
5. Update the TLB to include a new cache entry for the virtual address.

## Key Points About Virtual Memory

1. A process's physical address space does not have a consistent, physical memory ordering. Instead, processes have a virtual address space, which maps to their correspondingly allocated physical memory.
2. Within a process's virtual address space, there are different, distinct virtual memory areas (VMAs). These include separate VMAs for code, data, heap, stack, and other segments of the address space.
3. Different virtual memory areas can be backed by different storage types:
   - File-backed pages: For example, memory-mapped areas corresponding to shared libraries or memory-mapped files.
   - Anonymous pages: Used for private data like the heap or stack, initially not associated with any file.
4. Within virtual memory areas, data is not guaranteed to be physically contiguous, or even all physically present. For instance, a virtual memory area could be entirely swapped to disk, except for a single page which is currently in memory.
5. In many cases, particularly for file-backed VM areas, virtual addresses map contiguously to the backing storage from an initial offset. However, this isn't always true, as non-linear mappings are also possible.

## Virtual Memory Optimizations

The following are general optimizations employed by virtual memory that I may dig into the details of in the future:

1. The kernel page cache acts as an intermediate layer between file-backed pages and physical memory, improving I/O performance by caching frequently accessed file data.
2. Copy-on-Write (CoW) mechanisms allow multiple processes to share the same physical pages until one process needs to modify the data, optimizing memory usage for fork() operations.
3. Demand paging ensures that physical memory is allocated only when a page is actually accessed, not when it's initially mapped, improving memory efficiency.
4. The kernel can reclaim memory from processes by swapping out less frequently used pages, allowing for overcommitment of memory resources.
5. Memory protection is enforced at the page level, allowing fine-grained control over read, write, and execute permissions for different parts of a process's address space.
6. Huge pages allow the kernel to use larger page sizes (e.g., 2MB or 1GB instead of 4KB), reducing TLB pressure by requiring fewer entries to map the same amount of memory. This is particularly beneficial for applications with large working sets, like databases or JVM-based applications.

## Understanding /proc/[pid]/maps

The `/proc/[pid]/maps` file provides a detailed view of a process's memory mappings. Here's an example:

```
00400000-01b00000 r-xp 00000000 103:01 1705405  /path/to/executable
01d00000-01d01000 r--p 01700000 103:01 1705405  /path/to/executable
01d01000-01db0000 rw-p 01701000 103:01 1705405  /path/to/executable
2129c000-212bd000 rw-p 00000000 00:00 0         [heap]
7fd2934d5000-7fd293679000 r-xp 00000000 103:01 3066  /usr/lib64/libc-2.26.so
7fd293679000-7fd293878000 ---p 001a4000 103:01 3066  /usr/lib64/libc-2.26.so
7fd293878000-7fd29387c000 r--p 001a3000 103:01 3066  /usr/lib64/libc-2.26.so
7fff33b67000-7fff33b88000 rw-p 00000000 00:00 0 [stack]
7fff33bae000-7fff33bb0000 r-xp 00000000 00:00 0 [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0 [vsyscall]
```

Each line contains:
1. Address range (start-end)
2. Permissions:
   - r: readable
   - w: writable
   - x: executable
   - s: shared
   - p: private (copy-on-write)
3. Offset into the file
4. Device (major:minor)
5. Inode number
6. Pathname or [special] designation

## Shared Libraries and Memory

Shared libraries are a prime example of memory sharing in Linux. Looking at the libc mapping from our example:

```
7fd2934d5000-7fd293679000 r-xp 00000000 103:01 3066  /usr/lib64/libc-2.26.so
7fd293679000-7fd293878000 ---p 001a4000 103:01 3066  /usr/lib64/libc-2.26.so
7fd293878000-7fd29387c000 r--p 001a3000 103:01 3066  /usr/lib64/libc-2.26.so
7fd29387c000-7fd29387e000 rw-p 001a7000 103:01 3066  /usr/lib64/libc-2.26.so
```

Note the different sections:
- r-xp: Read and execute (code)
- r--p: Read-only data
- rw-p: Read-write data (private copy-on-write)

## The Heap and its Growth

The heap segment is used for dynamic memory allocation. In our example:

```
2129c000-212bd000 rw-p 00000000 00:00 0         [heap]
```

This shows a heap of approximately 132 KB (212bd000 - 2129c000 = 135168 bytes).

Important aspects of heap memory:
1. It can grow dynamically using `brk()` or `sbrk()`
2. Large allocations might use `mmap()` instead
3. Growth is not limited to the current size
4. The heap can become fragmented over time

## Memory Protection with mprotect

The `mprotect` system call allows dynamic changing of memory protection. Here's a complete example:

```c
#include <sys/mman.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

void protect_memory_region(void* addr, size_t len) {
    // Get page size and align address to page boundary
    size_t page_size = getpagesize();
    void* page_start = (void*)((uintptr_t)addr & ~(page_size - 1));
    
    // Set protection to read-only
    if (mprotect(page_start, page_size, PROT_READ) != 0) {
        int error = errno;
        printf("mprotect failed: %s\n", strerror(error));
        return;
    }
    
    printf("Memory protection set to read-only\n");
}
```

Protection remains in effect until:
1. Another `mprotect` call changes it
2. The memory is unmapped
3. The process terminates

## Shared Memory in Linux

Linux provides several mechanisms for shared memory. Here's an example using POSIX shared memory:

```c
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

void* create_shared_memory(const char* name, size_t size) {
    // Create shared memory object
    int fd = shm_open(name, O_CREAT | O_RDWR, 0666);
    if (fd == -1) {
        return NULL;
    }
    
    // Set the size
    if (ftruncate(fd, size) == -1) {
        close(fd);
        return NULL;
    }
    
    // Map it into memory
    void* ptr = mmap(NULL, size, 
                     PROT_READ | PROT_WRITE,
                     MAP_SHARED, fd, 0);
    
    close(fd);
    return (ptr == MAP_FAILED) ? NULL : ptr;
}

void* access_shared_memory(const char* name, size_t size) {
    int fd = shm_open(name, O_RDWR, 0666);
    if (fd == -1) {
        return NULL;
    }
    
    void* ptr = mmap(NULL, size,
                     PROT_READ | PROT_WRITE,
                     MAP_SHARED, fd, 0);
    
    close(fd);
    return (ptr == MAP_FAILED) ? NULL : ptr;
}
```

## How is mprotect enforced for a shared memory segment?

In the prior example, we used ```mprotect``` to change the memory protection level for portions
of a shared memory segment, which may be memory-mapped by other processes as well.

U

## Memory Ordering and mprotect

`mprotect` acts as a memory barrier, ensuring strict ordering of memory operations:

```c
// Example demonstrating memory ordering with mprotect
char* area = mmap(NULL, 4096, 
                  PROT_READ | PROT_WRITE,
                  MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

// Write to area
strcpy(area, "Hello");

// Make read-only - all previous writes are guaranteed to complete
mprotect(area, 4096, PROT_READ);

// Make writable again
mprotect(area, 4096, PROT_READ | PROT_WRITE);

// This write cannot be reordered before the previous mprotect
area[0] = 'X';
```

The kernel ensures:
1. All memory accesses complete before protection changes
2. Protection changes are fully visible before subsequent accesses
3. TLB flushes act as memory barriers
4. Memory operations cannot be speculated across protection changes

These are some virtual memory basics, I will probably supplement this in the future with more writeups about some of the following:
- Page tables and address translation
- Memory allocator internals
- NUMA memory management
- Transparent huge pages
- Memory policy and control groups