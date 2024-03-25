# Threads

* A thread is a unit of execution. It is similar to a process but it's lighter weight. Due to this
  reason, a thread is sometimes called a light-weight process.
* You can read more about this topic in the following chapters of
  [OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/). Keep in mind that OSTEP's discussions are much
  more in depth, which we consider out of scope for this course. Thus, you can use it as a reference
  but the main source should still be this lecture note.
    * [Chapter 26 Concurrency: An
      Introduction](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-intro.pdf).
    * [Chapter 27 Interlude: Thread API](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-api.pdf)

### Threads vs Processes

* Both threads and processes can execute in parallel on different cores. What are the differences
  then?
* Processes have separate (virtual) address spaces. If a process forks a child process, the child
  process gets its own address space.
* Threads share the same address space: text, data, bss, and heap segments.

  ```bash
  +-----------+
  | Kernel    |
  |           |
  |-----------|
  | Main      |
  | thread    |
  | stack     |
  |-----------|
  |           |
  |           |
  |           |
  |-----------|
  | Thread 1  |
  | stack     |
  |           |
  |-----------|
  |           |
  |           |
  |           |
  |           |
  |-----------|
  | Memory    |
  | mapping   |
  |-----------|
  |           |
  |           |
  |-----------|
  | Heap      |
  |           |
  |-----------|
  | BSS       |
  |-----------|
  | Data      |
  |-----------|
  | Text      |
  |-----------|
  |           |
  +-----------+
  ```

    * However, the stack is not shared across different threads (that are in the same process). Each
      thread gets its own stack.
    * Since creating a thread doesn't involve creating a new address space, it has less overhead
      than creating a process.
* One of the main reasons to use threads over processes is that threads share the same addresses
  hence data sharing is easy. For example, any thread can read from or write to a global variable.
* A process always has at least one thread called the main thread.

### POSIX Threads

* POSIX threads (pthreads) is a popular interface used for thread management.
* `man pthreads` explains pthreads in detail, including what exactly is shared across threads and
  what is not.
* There are a few common functions that programs often use.
    * `pthread_create()`
        * `man pthread_create`
        * `pthread_t`: this is the type used for thread IDs.
        * `pthread_create()` expects a function pointer to a thread function (that will run inside a
          thread).
        * It also expects `void *arg`, which gets passed to the thread function as an argument when
          the thread starts. Since this is `void *`, you can cast any pointer. If you want to pass
          multiple arguments, you can create a `struct` and pass the pointer to it.
        * `pthread_attr_t` specifies various attributes of the new thread.
    * `pthread_exit()` terminates the calling thread. Performing a return from the thread function
      implicitly called `pthread_exit()` with its return value.
    * `pthread_self()` returns the caller's id.
    * `pthread_join()` waits until the thread identified by the parameter terminates.
    * `pthread_detach()` lets the calling thread just run. We can use this when we don't need to
      return anything.

### Pthread Activity

* Write a program where the main thread creates another thread. In the main thread, it should print
  out the new thread's ID, wait until it terminates, and print out the return value. The other
  thread should get a string from the main thread as the argument, print that out as well as its own
  ID, and return the length of the received string.
    * Note: a thread start function can cast the return type to integer types (e.g., `long`).
    * When compiling a program that uses pthreads, you need to use `-pthread` as a compiler option,
      e.g., `clang -pthread example.c`.

### Data Race Activity & Problem

* Activity: write a program that has two additional threads. Define a global variable called `cnt`
  and initialize it to 0. Two new threads should each run a loop 10,000,000 times that keeps
  incrementing `cnt`. The main thread should wait on both threads, and when everything's done,
  should print out the value of `cnt`. Run it multiple times and see the outputs.
    * The outputs should be *non-deterministic*, meaning the behavior should be different every time
      the program runs.
    * This is as oppose to *deterministic* behavior where a program behaves the same way every time
      it runs.
    * The execution of a concurrent program is typically non-deterministic.
* This problem is known as the *data race problem*.
    * If you run multiple threads on different cores, they run at the same time.
    * If you perform an addition in a C program, it appears to be a single operation but it is not.
    * Adding an integer to a variable consists of multiple steps roughly as follows.
        * Load the variable's data from memory.
        * Perform the addition.
        * Store the result back to memory.
        * (There are more steps in reality, but this is enough for our discussion.)
    * Thus, if you perform an addition on a global variable in two different threads, they will each
      perform the above steps at the same time. Thus, the following scenario could happen.
        * Assume we are dealing with a variable (`cnt`) that currently contains 5 as its data.
        * Assume that each thread adds 1 to `cnt` and assigns the result back to `cnt`, just like
          the code from the above activity.
        * Since two threads can run concurrently, the following could happen.
            * Thread0 loads `cnt`'s data (5) from memory.
            * At the same time, Thread1 loads `cnt`'s data (5) from memory.
            * Thread0 adds 1 to 5.
            * At the same time, Thread1 adds 1 to 5.
            * Thread0 write the result back to memory, i.e., it writes 6 back to memory.
            * At the same time, Thread1 write the result back to memory, i.e., it writes 6 back to
              memory.
        * The correct result that we expect is 7 since we have added 1 twice to `cnt`. However,
          `cnt`'s final value in memory is 6, not 7, since Thread0 and Thread1 both wrote 6 back to
          memory.
    * This is called the data race problem, where different threads *race* to update data and
      overwrite each other's result.
    * More generally, a *race condition* is a condition in which the correctness of a program
      depends on the timing and/or order of operations. Between a data race and a race condition,
      there is significant overlap but they are not the same thing. We don't get into the details in
      this course, but if you are interested, you can [read more about
      it](https://blog.regehr.org/archives/490).
