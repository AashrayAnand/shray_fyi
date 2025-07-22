# Using ```mprotect``` For Memory Protection

I recently took some notes when looking into the linux ```mprotect``` interface, which enables setting the protection on a region of [memory](https://linux.die.net/man/2/mprotect). In particular, this exercise was related to how ```mprotect``` interacts with memory-mapped shared memory segments.

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

We can refer to the key concepts I went over before in [Refreshing Virtual Memory Concepts](./virtual_memory_1.md) to understand how applying controls on a shared memory segment would affect the underlying processes. Although multiple processes may memory map the same shared memory segment, they each have a distinct virtual memory area that corresponds to this memory mappings, which are controlled independently of one another.

Therefore, although we may ```mprotect``` a virtual address that correlates to shared memory, we will only end up enforcing the protection level on ourself, rather than other processes accessing the same shared memory.

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