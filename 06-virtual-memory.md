# Virtual Memory

* Virtual memory is one of the most important OS concepts.
* It is also a good example that shows the power of abstraction.
* You can read more about this topic in the following chapters of
  [OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/). Keep in mind that OSTEP's discussions are much
  more in depth, which we consider out of scope for this course. Thus, you can use it as a reference
  but the main source should still be this lecture note.
    * [Chapter 13 The Abstraction: Address
      Spaces](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-intro.pdf)
    * [Chapter 15 Mechanism: Address
      Translation](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-mechanism.pdf)
    * [Chapter 18 Paging: Introduction](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-paging.pdf)
    * [Chapter 16 Segmentation](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-segmentation.pdf)

## Memory Layout in the Early Days

* The entire memory was divided into two: OS and program.

  ```bash
  +-----------+
  |           |
  | OS        |
  |           |
  |-----------|
  |           |
  | Process   |
  |           |
  |           |
  |           |
  +-----------+
  ```

* Only a single program could run.
* However, there were many users who wanted to run their own programs.
* Users could not share the machine, leading to low utilization.

## Early Memory Sharing Attempt

* Memory was divided into fixed-size regions.

  ```bash
  +-----------+
  | OS        |
  |-----------|
  |           |
  | Process A |
  |           |
  |-----------|
  |           |
  | Free      |
  |           |
  |-----------|
  |           |
  | Process C |
  |           |
  |-----------|
  |           |
  | Process B |
  |           |
  +-----------+
  ```

* We could run multiple processes.
* Each process used a range of fixed addresses.
* Problems
    * Each process could only use a small size memory region.
    * There was no protection/isolation across processes. A pointer from one program could access
      other programs memory locations.

## Solution: Virtual Memory

* "All problems in computer science can be solved by another level of indirection, except for the
  problem of too many levels of indirection." -- David Wheeler
* Virtual memory is a mechanism to enable (i) physical memory sharing for multiple processes,
  and (ii) isolation of each process's memory access
* Simply put, a process uses *virtual* memory instead of physical memory.
* Virtual memory consists of two components, virtual address space and address translation.
* Virtual memory is a good example that demonstrates the power of *abstractions*.

### Virtual Address Space

* It is an abstraction that gives the illusion that a program/process is using the entire address
  space.
* Each program/process gets its own address space that ranges from 0 to the address max allowed by
  the underlying CPU architecture, e.g., 2^64 - 1.
* Each address in a virtual address space is called a *virtual address* (as opposed to a *physical
  address* that points to a physical memory location).
* The following is an example with three processes:

  ```bash
  Process A's        Process B's        Process C's
  address space      address space      address space

  +-----------+      +-----------+      +-----------+
  | Kernel    |      | Kernel    |      | Kernel    | (Virtual) Address Max
  |           |      |           |      |           | (e.g., 2^64 - 1)
  |-----------|      |-----------|      |-----------|
  | Stack     |      | Stack     |      | Stack     |
  |           |      |           |      |           |
  |-----------|      |-----------|      |-----------|
  |           |      |           |      |           |
  |-----------|      |-----------|      |-----------|
  | Memory    |      | Memory    |      | Memory    |
  | mapping   |      | mapping   |      | mapping   |
  |-----------|      |-----------|      |-----------|
  |           |      |           |      |           |
  |           |      |           |      |           |
  |-----------|      |-----------|      |-----------|
  | Heap      |      | Heap      |      | Heap      |
  |           |      |           |      |           |
  |-----------|      |-----------|      |-----------|
  | BSS       |      | BSS       |      | BSS       |
  |-----------|      |-----------|      |-----------|
  | Data      |      | Data      |      | Data      |
  |-----------|      |-----------|      |-----------|
  | Text      |      | Text      |      | Text      |
  |-----------|      |-----------|      |-----------|
  |           |      |           |      |           |
  |           |      |           |      |           | (Virtual) Address 0
  +-----------+      +-----------+      +-----------+
  ```

* Each program/process only sees its own address space, i.e., it does not see any other process's
  address space, providing memory isolation among different processes.
* A user-level program/process *only* deals with virtual addresses and never with physical
  addresses. Kernel-level components can deal with both virtual addresses and physical addresses.

### Address Translation

* An address translation mechanism enables a process to use a virtual address to access a physical
  address.
* It divides each process's virtual address space into smaller regions and map those regions to
  physical memory. More specifically, the kernel keeps track of which virtual memory region is
  mapped to which physical memory region (for each process). When a process accesses a virtual
  memory address (using a pointer, for example), the kernel (i) finds which virtual memory region it
  belongs to, (ii) also finds which physical memory region it is mapped to, and (iii) redirects the
  access to the correct physical memory region and the address within it. This way, the process can
  read or write data to physical memory. This is called *address translation*, i.e., a virtual
  address is translated into a physical address to read or write data.

  ```bash
  Process A's         Physical            Process B's
  address space       Memory              address space

  +-----------+       +-----------+       +-----------+
  |           | --+   |           | <---- |           |
  |-----------|   |   |-----------|       |-----------|
  |           |   +-> |           |       |           |
  |-----------|       |-----------|       |-----------|
  |           |       |           |   +-- |           |
  |-----------|       |-----------|   |   |-----------|
  |           |   +-> |           |   |   |           |
  |-----------|   |   |-----------|   |   |-----------|
  |           |   |   |           | <-+   |           |
  |-----------|   |   +-----------+       |-----------|
  |           | --+                       |           |
  |-----------|                           |-----------|
  |           |                           |           |
  |-----------|                           |-----------|
  |           |                           |           |
  |-----------|                           |-----------|
  |           |                           |           |
  |-----------|                           |-----------|
  |           |                           |           |
  |-----------|                           |-----------|
  |           |                           |           |
  +-----------+                           +-----------+
  ```

* There are two popular approaches for address translation, paging and segmentation.
    * The basic difference is how to divide a virtual address space into smaller regions.
    * Paging uses fixed-size regions (called *pages*), while segmentation uses meaningful regions
      (called *segments*) with varying sizes.

### Paging

* A virtual address space is divided into fixed-size *pages*. 4 KB is a popular size but modern OSs
  have bigger pages (e.g., 4 MB) as well. For example, suppose the virtual address space size is 16
  KB and the page size is 4 KB. The virtual address space is then divided into four pages. The
  following is an example with two processes (where page numbers are in binary).

  ```bash
  Process A's      Process B's
  address space    address space

  +---------+      +---------+
  | page 11 |      | page 11 |
  |---------|      |---------|
  | page 10 |      | page 10 |
  |---------|      |---------|
  | page 01 |      | page 01 |
  |---------|      |---------|
  | page 00 |      | page 00 |
  +---------+      +---------+
  ```

* Physical memory is also divided into *page frames* that has the same size as pages. For example,
  suppose we have physical memory of size 8 KB and the page size is 4 KB. The physical memory is
  divided into two page frames (where the page frame numbers are in binary).

  ```bash
  Physical Memory

  +---------------+
  | page frame 01 |
  |---------------|
  | page frame 00 |
  +---------------+
  ```

* Address translation
    * A virtual address is divided into two parts: <page number, offset>.
    * Example: 6-bit virtual address space divided into 2-bit page numbers and 4-bit offsets
        * Address 100101 is divided into page number 10 and offset 0101.
        * Address 000010 is divided into page number 00 and offset 0010.
    * When a process accesses a (virtual) address, paging translates the address's page number to a
      page frame number (as describe next). The offset does not change.
    * The kernel maintains a *page table* per process. Each page table contains mappings of a page
      number (virtual) to a page frame number (physical). The following is an example where page 00
      is mapped to page frame 01 and page 10 is mapped to page frame 00.

      ```bash
      Page table

      +---------------------------------+
      | Page number | Page frame number |
      |-------------|-------------------|
      | 00          | 01                |
      |-------------|-------------------|
      | 10          | 00                |
      +---------------------------------+

      Virtual            Physical
      Address            Memory
      Space
                   +--------------------+
      +---------+  |     +----------+   |
      | page 11 |  | +-> | frame 01 |   |
      |---------|  | |   |----------|   |
      | page 10 | -+ |   | frame 00 | <-+
      |---------|    |   +----------+
      | page 01 |    |
      |---------|    |
      | page 00 | ---+
      +---------+
      ```

    * In the above example, when the process accesses (virtual) address 000001, it is translated
      into 010001. In other words, the first two bits (00, i.e., the page number) is translated into
      01 (the mapped page frame number) and the offset (0001) doesn't change. When the process
      access address 101010, it is translated into 001010. In other words, the first two bits (10)
      is translated into 00, according to the page table above.
    * This way, a process can access physical memory (indirectly via virtual memory).
    * Since there are many more (virtual) pages than (physical) page frames, we cannot map all pages
      to page frames at the same time. Thus, we map pages to page frames when a process accesses
      them. We discuss more details later.
    * There is hardware support for address translation to make it fast. In other words, the kernel
      and hardware jointly performs address translation.
* Page table size:
    * If page numbers use *n* bits, the maximum possible number of pages is *2^n*.
    * If offsets use *m* bits, the maximum possible page size is *2^m - 1*.
    * For example, suppose the page size is 4 KB (i.e., *m* is 12) and we use a 32-bit architecture.
      *n* is then 20 (32 - 12), which means that we can have *2^20* pages. This is 1M pages and the
      page table should be able to handle this many pages. There are many solutions that exist to
      handle a large number of pages in a page table, such as hierarchical page tables, which we
      don't get to in this course.
* Modern OSs typically use paging rather than segmentation.

### Segmentation

* Segmentation is similar to paging in the sense that it divides a virtual memory space into smaller
  regions and uses address translation to enable processes to access physical memory via virtual
  memory. The basic difference is that segmentation does not use fixed-size regions, but meaningful
  regions called *segments*.
* Segments are meaningful regions that programmers can recognize.
    * The text segment, the data segment, the stack, the heap, etc. are all examples of segments.
      There can be other segments that one can define. Nonetheless, unlike a page, which is just a
      sequence of bytes, each segment is a region with a meaning that a programmer understands.
* Address translation
    * Similar to paging, a virtual address is divided into two parts: <segment number, offset>.
    * However, segment sizes can be different unlike paging. Thus, there are additional
      considerations for address translation, which we won't get into in this course.
* Segmentation suffers from *external fragmentation*.
    * Since segments are of different sizes, we might run into a scenario where we have left with
      many small regions that can't fit a large segment, even if the total sum of all free regions
      exceed the size of the large segment. In the example below, if we need to map a segment of
      size 42 KB, we cannot do it even if the total free space is 72 KB. This is called external
      fragmentation.

      ```bash
      Physical Memory

      +-------------------+
      | Used by a segment |
      |                   |
      |                   |
      |-------------------|
      | Free (16 KB)      |
      |-------------------|
      | Used by a segment |
      |                   |
      |                   |
      |                   |
      |                   |
      |-------------------|
      | Free (24 KB)      |
      |                   |
      |-------------------|
      | Used by a segment |
      |                   |
      |                   |
      |                   |
      |                   |
      |                   |
      |                   |
      |-------------------|
      | Free (32 KB)      |
      |                   |
      |                   |
      |                   |
      |                   |
      +-------------------+
      ```

    * Paging does not have this problem since every page is of the same size.
    * On the other hand, paging might have *internal fragmentation* problems. When a program uses a
      page, it may not use the whole page but only a small portion of the page. Nevertheless, the
      unit of allocation for paging is pages (i.e., paging swaps a page in or out). Thus, there can
      be wasted space internal to a page. This is called *internal fragmentation*.

## What If We Run Out of Memory?

* We have limited physical memory but the size of a modern virtual memory space is vast. We can't
  bring in all virtual pages/segments to physical memory. What do we do?
* Demand paging & swapping
    * Demand paging: a page is brought into memory only when needed
    * Swapping: if we don't have space to bring in a new page, kick out a page already in memory to
      disk and bring in the new page.
    * Swap space: disk space dedicated to store swapped-out pages.
    * How do we decide which memory page to swap out? We need a *page replacement* algorithm
      (discussed later).
* Why does this work?
    * Insight: A typical program only accesses a small portion of its memory space.
    * This is based on *locality* of access.
        * Temporal locality: if a program accesses a memory location, it is likely that it's going
          to access it again in the near future.
        * Spatial locality: if a program accesses a memory location, it is likely that it's going to
          access other memory locations nearby.
    * Paging/segmentation leverage spatial locality because, when a memory location is accessed, it
      brings in the region that the location belongs to, not just the specific memory location.
    * Demand paging & swapping leverage temporal locality as it brings memory regions in and out of
      memory as accessed.

## Page Replacement Algorithms

* The question is when the memory is full (i.e., all page frames are used) and we need to bring in a
  new page, which page do we swap out from memory to disk?
* *Page fault*: when a memory location is accessed but the corresponding page is not found in
  physical memory, we say that a *page fault* has occurred. In this case, we need to bring in
  the right page to memory. If memory is full and there is no space left for a new page, we run a
  page replacement algorithm to swap out a memory page (from memory to disk) and make room.

### The Optimal Page Replacement Algorithm

* The optimal page replacement algorithm picks the page that will not be used in the future at all
  or that will be used last among all pages currently in memory.
* This assumes that we know the future, which is impossible. Thus, this is only a theoretical
  exercise.
* Example
    * Suppose our memory has 4 page frames.
    * Suppose memory page access occurs like this (by page number): 1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4,
      5
    * The page frames will be used like the following according to the optimal page replacement
      algorithm. * indicates a page replacement.

      ```bash
                                                       *             *
      Page access:  1,      2,      3,      4,  1, 2,  5,  1, 2, 3,  4,  5
                  +---+   +---+   +---+   +---+      +---+         +---+
                  | 1 |   | 1 |   | 1 |   | 1 |      | 1 |         | 4 |
                  |---|   |---|   |---|   |---|      |---|         |---|
                  |   |   | 2 |   | 2 |   | 2 |      | 2 |         | 2 |
                  |---|   |---|   |---|   |---|      |---|         |---|
                  |   |   |   |   | 3 |   | 3 |      | 3 |         | 3 |
                  |---|   |---|   |---|   |---|      |---|         |---|
                  |   |   |   |   |   |   | 4 |      | 5 |         | 5 |
                  +---+   +---+   +---+   +---+      +---+         +---+
      ```

        * Accessing page 5 is when we run the replacement algorithm. Page 4 is used last in the
          future, so we swap it out.
        * Later, accessing page 4 also triggers the replacement algorithm. Pages 1, 2, and 3 are not
          used in the future, so we can swap out any one of those.
    * There are 6 page faults.
* Page replacement algorithms try to approximate this as much as possible.

### FIFO (First In, First Out)

* This algorithm keeps track of when a page was brought in to memory. The first one that was brought
  in gets swapped out first.
* Using the same page access sequence as above,

  ```bash
                                                   *       *       *       *       *       *
  Page access:  1,      2,      3,      4,  1, 2,  5,      1,      2,      3,      4,      5
              +---+   +---+   +---+   +---+      +---+   +---+   +---+   +---+   +---+   +---+
              | 1 |   | 1 |   | 1 |   | 1 |      | 5 |   | 5 |   | 5 |   | 5 |   | 4 |   | 4 |
              |---|   |---|   |---|   |---|      |---|   |---|   |---|   |---|   |---|   |---|
              |   |   | 2 |   | 2 |   | 2 |      | 2 |   | 1 |   | 1 |   | 1 |   | 1 |   | 5 |
              |---|   |---|   |---|   |---|      |---|   |---|   |---|   |---|   |---|   |---|
              |   |   |   |   | 3 |   | 3 |      | 3 |   | 3 |   | 2 |   | 2 |   | 2 |   | 2 |
              |---|   |---|   |---|   |---|      |---|   |---|   |---|   |---|   |---|   |---|
              |   |   |   |   |   |   | 4 |      | 4 |   | 4 |   | 4 |   | 3 |   | 3 |   | 3 |
              +---+   +---+   +---+   +---+      +---+   +---+   +---+   +---+   +---+   +---+
  ```

* 10 page faults
* This is simple but does not consider useful properties like locality.

### LRU (Least Recently Used)

* This algorithm replaces page that has not been used for longest period.
    * It tries to approximate the optimal algorithm.
    * It tries to infer the future based on past.
* Using the same page access sequence,

  ```bash
                                                   *           *       *       *
  Page access:  1,      2,      3,      4,  1, 2,  5,  1, 2,   3,      4,      5
              +---+   +---+   +---+   +---+      +---+       +---+   +---+   +---+
              | 1 |   | 1 |   | 1 |   | 1 |      | 1 |       | 1 |   | 1 |   | 5 |
              |---|   |---|   |---|   |---|      |---|       |---|   |---|   |---|
              |   |   | 2 |   | 2 |   | 2 |      | 2 |       | 2 |   | 2 |   | 2 |
              |---|   |---|   |---|   |---|      |---|       |---|   |---|   |---|
              |   |   |   |   | 3 |   | 3 |      | 5 |       | 5 |   | 4 |   | 4 |
              |---|   |---|   |---|   |---|      |---|       |---|   |---|   |---|
              |   |   |   |   |   |   | 4 |      | 4 |       | 3 |   | 3 |   | 3 |
              +---+   +---+   +---+   +---+      +---+       +---+   +---+   +---+
  ```

* 8 page faults
* Keeping track of access time is not simple to implement. The next algorithm approximates this.

### Second-Chance

* This algorithm is an approximation of LRU.
    * Each page has a reference bit (ref_bit), initially = 0
    * When a page is accessed, we set ref_bit to 1.
    * We maintain a moving pointer to the next (candidate) victim.
    * When choosing a page to replace, check ref_bit of victim:
        * If ref_bit == 0, replace it.
        * Else set ref_bit to 0.
            * Leave page in memory (give it another chance).
            * Move pointer to next page.
            * Repeat till a victim is found.

* Example
    * Assume the following state before running the algorithm. We move the next victim pointer down.
      If it reaches the bottom, we move the pointer to the top in a circular fashion.

      ```bash

                  reference bits        pages

                                      +-------+
                      +---+           |       |
                      | 0 |           |       |
                      +---+           |       |
                                      +-------+

                                      +-------+
                      +---+           |       |
      next victim --> | 1 |           |       |
                      +---+           |       |
                                      +-------+

                                      +-------+
                      +---+           |       |
                      | 1 |           |       |
                      +---+           |       |
                                      +-------+

                                      +-------+
                      +---+           |       |
                      | 0 |           |       |
                      +---+           |       |
                                      +-------+
      ```

    * If we need to replace a page, we look at the next victim pointer and check ref_bit according
      to the algorithm. If it is 0, we replace it. In the above case, it is not 0, so we set ref_bit
      to 0 and move the pointer down as follows.

      ```bash

                  reference bits        pages

                                      +-------+
                      +---+           |       |
                      | 0 |           |       |
                      +---+           |       |
                                      +-------+

                                      +-------+
                      +---+           |       |
                      | 0 |           |       |
                      +---+           |       |
                                      +-------+

                                      +-------+
                      +---+           |       |
      next victim --> | 1 |           |       |
                      +---+           |       |
                                      +-------+

                                      +-------+
                      +---+           |       |
                      | 0 |           |       |
                      +---+           |       |
                                      +-------+
      ```

    * We look at the next victim and ref_bit is still not 0. We set it to 0 and move the pointer
      down as follows.

      ```bash

                  reference bits        pages

                                      +-------+
                      +---+           |       |
                      | 0 |           |       |
                      +---+           |       |
                                      +-------+

                                      +-------+
                      +---+           |       |
                      | 0 |           |       |
                      +---+           |       |
                                      +-------+

                                      +-------+
                      +---+           |       |
                      | 0 |           |       |
                      +---+           |       |
                                      +-------+

                                      +-------+
                      +---+           |       |
      next victim --> | 0 |           |       |
                      +---+           |       |
                                      +-------+
      ```

    * Now the next victim's ref_bit is 0. We replace it with a new page.

## Thrashing

* If you run a process that accesses a large amount of memory, you might run into a situation where
  you need to keep bringing in new pages and keep replacing existing pages.
* Thrashing: a process is in a situation where it is too busy swapping in and out pages and not
  really executing its program on the CPU.
