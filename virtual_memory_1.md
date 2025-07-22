# A Guide to Key Concepts of Linux Virtual Memory Areas

## Table of Contents
1. [Introduction to Virtual Memory](#introduction-to-virtual-memory)
2. [Memory Mapping in Linux](#memory-mapping-in-linux)
3. [Understanding /proc/[pid]/maps](#understanding-procpidmaps)
4. [Shared Libraries and Memory](#shared-libraries-and-memory)
5. [The Heap and its Growth](#the-heap-and-its-growth)
6. [Memory Protection with mprotect](#memory-protection-with-mprotect)
7. [Shared Memory in Linux](#shared-memory-in-linux)
8. [Memory Ordering and mprotect](#memory-ordering-and-mprotect)

## Introduction to Virtual Memory

Virtual memory is a memory management technique that provides an abstraction of the storage resources available to a program. It maps memory addresses used by a program, called virtual addresses, into physical addresses in computer memory.

Key benefits of virtual memory include:
- Process isolation
- Efficient memory utilization
- Ability to use more memory than physically available through paging
- Memory protection between processes
- Simplified memory allocation

## Memory Mapping in Linux

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