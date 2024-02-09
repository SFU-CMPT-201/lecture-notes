# Memory Management

* This is mainly about heap memory allocation and deallocation.
    * Typically, we call `malloc()` or `calloc()`, and `free()`.
    * However, we will learn how these can be implemented.
    * This is a good exercise to deepen our understanding for low-level programming manipulating
      memory (and every systems programming course does this).
* We are not talking about physical memory here. User processes can only use virtual memory, not
  physical memory.
* You can read more about this topic in the following chapters of
  [OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/). Keep in mind that OSTEP's discussions are much
  more in depth, which we consider out of scope for this course. Thus, you can use it as a reference
  but the main source should still be this lecture note.
    * [Chapter 13 The Abstraction: Address
      Spaces](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-intro.pdf)
    * [Chapter 14 Interlude: Memory API](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-api.pdf)
    * [Chapter 15 Free-Space Management](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-freespace.pdf)

## Prerequisites

This lecture assumes that you know the following already.

* How to use `malloc()` and `free()` in C and understand what they do.
* How to implement a singly- and doubly-linked list in C.
* The stack and the heap, and how a program's variables use those in C and how they are located in
  them.

## `brk()` and `sbrk()`

* `brk()` and `sbrk()` allow us to increase the heap size by moving the *program break*.
* `man sbrk`
    * "the program break is the first location after the end of the uninitialized data segment."
    * "Increasing the program break has the effect of allocating memory to the process; decreasing
      the break deallocates memory."
    * Another way to allocate more memory in a process is using `mmap()` which we will discuss in a
      later lecture.
* Though we can just use `sbrk()` to allocate more memory to our process, the call takes long since
  it's a system call that crosses the user-kernel boundary. Thus, it's better to get a chunk of
  memory and manage the chunk piece-by-piece as necessary.
    * `malloc()` calls `sbrk()` to increase the heap size and gives out piece-by-piece as requested
      by a `malloc()` call.
* Question arises: how do you manage the chunk?
    * Allocation strategies
    * Deallocation strategies

## Memory Allocation

* A memory allocator manages the heap. As a program requests memory allocation, it marks a region
  inside the heap as "allocated" and returns the pointer to the beginning of that region to the
  program.
* In order to do that, the memory allocator needs to keep track of which part of the heap is not
  used.
* During execution, a program requests heap allocation and frees allocated regions. Thus, the heap
  can be *fragmented*, i.e., some regions are allocated while other regions are freed.

  ```bash
  Heap
  +-----------+
  | Used      |
  |-----------|
  |           |
  | Freed     |
  |           |
  |-----------|
  |           |
  | Used      |
  |           |
  |-----------|
  |           |
  | Used      |
  |           |
  |-----------|
  |           |
  | Freed     |
  |           |
  +-----------+
  ```

* In order to keep track of unused regions (or *blocks*) of the (fragmented) heap, we can use a
  linked list of free blocks. We will later discuss this further.
* For the sake of our discussion, let's assume that we have a linked list of free blocks.
* Allocating memory (e.g., serving a `malloc()` call) requires us to find a big enough memory block
  that can satisfy the request.
* There are three strategies we can consider.
    * First-fit
        * We traverse the linked list of free blocks, and allocates the first block that can fit the
          requested size.
        * We keep the remaining free space free for later requests.
        * An advantage of this is its implementation simplicity. Also, it is fast as it only needs
          to find the first big enough block.
        * A disadvantage is that sometimes it pollutes the beginning of the free list with small
          objects, leading to more search time for bigger allocation requests.
    * Best-fit
        * We traverse the linked list and allocate the smallest free block that is big enough.
        * We keep the remaining free space free for later requests.
        * An advantage of this is that it reduces wasted memory space.
        * A disadvantage is that it must search the entire list (unless ordered by size which has
          additional implementation complexity).
        * Another disadvantage is that it may create many small free blocks, leading to more chances
          of *external fragmentation* (explained below).
    * Worst-fit
        * We traverse the linked list and allocate the largest free block.
        * Ad advantage is that it produces large leftover free blocks.
        * A disadvantage is that it must search the entire list.
* External fragmentation
    * We might run into a scenario where we have left with many small blocks that can't satisfy a
      large allocation request, even if the total sum of all free blocks exceed the size of the
      large allocation. This is called external fragmentation.
    * There's a similar problem called *internal* fragmentation. We will discuss it in [Virtual
      Memory](06-virtual-memory.md)

## Memory Deallocation

* Once a program is done with a memory block, it frees the memory block, i.e., it returns the memory
  block to the memory allocator. The memory allocator then makes this block available again to
  satisfy future allocation requests.
* `free()` implementation
    * We can use a linked list of free blocks.
    * The head points to the most recently freed memory.
    * We use a header for each free block to keep track of the size of the free block and a pointer
      to the next free block.
* Below is an example with the heap size of 256 bytes. Let's assume that `size` and `next` are 8
  bytes each.

  ```bash
  Initial state:
           +------------+
           |            | 255
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |   Free     |
           |            |
           |            | 16
           |------------|
           |(Header)    | 15
           |size = 240  |    (size = total_free - header_size = 256 - 16 = 240)
  head --> |next = null | 0
           +------------+
  ```

  ```bash
  After allocating 100 bytes (we *split* an existing free block):
           +------------+
           |            | 255
           |   Free     |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |------------|
           |(Header)    | 131
           |size = 124  |    (size = 256 - 116 - 16 = 124)
  head --> |next = null | 116
           |------------|
           |            | 115
           | Allocated  |
           |            | 16
           |------------|
           |(Header)    | 15
           |size = 100  |
           |next = null | 0
           +------------+
  ```

  ```bash
  After allocating another 50 bytes (again, we split):
           +------------+
           |            | 255
           |   Free     |
           |            |
           |            |
           |            | 198
           |------------|
           |(Header)    | 197
           |size = 58   |    (size = 256 - 182 - 16)
  head --> |next = null | 182
           |------------|
           |            | 181
           | Allocated  |
           |            | 132
           |------------|
           |(Header)    | 131
           |size = 50   |
           |next = null | 116
           |------------|
           |            | 115
           | Allocated  |
           |            | 16
           |------------|
           |(Header)    | 15
           |size = 100  |
           |next = null | 0
           +------------+
  ```

  ```bash
  Freeing the first 100-byte chunk (the freed block gets inserted at the list head):
           +------------+
           |            | 255
           |   Free     |
           |            |
           |            |
           |            | 198
           |------------|
           |(Header)    | 197
           |size = 58   |
           |next = null | 182
           |------------|
           |            | 181
           | Allocated  |
           |            | 132
           |------------|
           |(Header)    | 131
           |size = 50   |
           |next = null | 116
           |------------|
           |            | 115
           | Free       |
           |            | 16
           |------------|
           |(Header)    | 15
           |size = 100  |
  head --> |next = 182  | 0 (next points to the previous head at 182)
           +------------+
  ```

  ```bash
  Freeing the next 50-byte chunk (the freed block gets inserted at the list head):
           +------------+
           |            | 255
           |   Free     |
           |            |
           |            |
           |            | 198
           |------------|
           |(Header)    | 197
           |size = 58   |
           |next = null | 182
           |------------|
           |            | 181
           | Free       |
           |            | 132
           |------------|
           |(Header)    | 131
           |size = 50   |
  head --> |next = 0    | 116  (next points to the previous head at 0)
           |------------|
           |            | 115
           | Free       |
           |            | 16
           |------------|
           |(Header)    | 15
           |size = 100  |
           |next = 182  | 0
           +------------+
  ```

* Avoiding external fragmentation using *coalescing*: if there are consecutive free blocks, we
  combine them into a larger block. This makes room for larger allocation requests.

  ```bash
  After coalescing:
           +------------+
           |            | 255
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           |            |
           | Free       |
           |            | 16
           |------------|
           |(Header)    | 15
           |size = 240  |
  head --> |next = null | 0
           +------------+
  ```
