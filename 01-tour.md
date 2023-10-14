# A Tour of Computer Systems

* We're mainly discussing the *terminology* necessary for the course and generally
  for computer systems.
    * The terms that you will encounter in this course as well as any other contexts related to
      systems.
* The traditional stack

  ```bash
  +--------------+
  | applications |
  |--------------|  <-- syscall interface (the focus of the course)
  | kernel       |
  |--------------|
  | hardware     |
  +--------------+
  ```

* Applications
    * Systems programming is low-level programming that directly interacts with hardware or the OS.
        * Only a few low-level languages, e.g., C, C++, Rust, give you the ability to do systems
          programming, e.g., raw memory access. E.g., Python and Java don't allow you to do that.
    * Higher-level programs don't typically need a systems programming language, unless it needs
      high performance.
    * We need to choose a language that fits the target program's goals.
* Virtualization: there are different types of virtualization.
    * VMM (virtual machine monitor) directly atop hardware: VMM emulates hardware for each VM.

      ```bash
        VM1          VM2          VM3
      +--------+   +--------+   +--------+
      | Apps   |   | Apps   |   | Apps   |
      +--------+   +--------+   +--------+
      | Kernel |   | Kernel |   | Kernel |
      +--------+---+--------+---+--------|
      | VMM                              |
      |----------------------------------|
      | hardware                         |
      +----------------------------------+
      ```

    * VMM atop the kernel: one VMM for each VM, emulating hardware.

      ```bash
        VM1          VM2          VM3
      +--------+   +--------+   +--------+
      | Apps   |   | Apps   |   | Apps   |
      +--------+   +--------+   +--------+
      | Kernel |   | Kernel |   | Kernel |
      +--------+   +--------+   +--------|
      | VMM    |   | VMM    |   | VMM    |
      +--------+---+--------+---+--------|
      | Kernel                           |
      |----------------------------------|
      | hardware                         |
      +----------------------------------+
      ```

    * Containerization: combines many features that Linux provides for isolation. This includes
      process isolation (namespaces), resource control/isolation (cgroups), etc. No need to run
      another kernel.

      ```bash
        Container1          Container2          Container3
      +---------------+   +---------------+   +---------------+
      | Apps          |   | Apps          |   | Apps          |
      +---------------+   +---------------+   +---------------+
      | containerizer |   | containerizer |   | containerizer |
      +---------------+---+---------------+---+---------------|
      | Kernel                                                |
      |-------------------------------------------------------|
      | hardware                                              |
      +-------------------------------------------------------+
      ```

* Hardware
    * (Look at [hardware-figures.pptx](hardware-figures.pptx) together for figures.)
    * Components: CPU, mem, and I/O devices.
    * Two fundamental components in computing: computation and data
    * This is captured in the current computer architecture in CPU and memory.
    * The evolution of CPU
        * Moore's Law
        * Originally, CPU was what is now known as single core, that could only run one sequence of
          instructions.
        * The focus was speeding up single core until ~2000 (Moore's Law
          graph). Then around 2005, CPU designers shifted focus to adding more cores, i.e., running
          multiple sequences of code at the same time.
    * The evolution of memory
        * It doesn't matter how fast a CPU is since it has to go get the data from memory.
        * How the speed gap was addressed:
            * First, registers (very small memory inside a CPU)
            * Then CPU designers were able to pack more and more things within the size size.
            * Then they added cache (larger but still small memory inside a CPU).
            * Then they added more caches. L1, L2, and L3.
    * Hardware threading
        * A technique that makes a single core appear as multiple cores (threads)
    * NUMA (Non-Uniform Memory Access)
        * Multiple CPUs (sockets)
        * Memory access time is non-uniform based on where the CPU is and where the memory is
          (farther away == takes more time to access).
    * Memory hierarchy
        * Registers, cache, mem, SSDs, hard drives, tapes.
            * Registers are the smallest in size but the closest to the CPU (fastest).
            * Tapes are the slowest but the largest in size.
        * Trade-offs
            * Bigger size == less expensive (size correlates with price)
            * Distance (access speed): faster == closer to CPU
            * Persistence
                * "commit"---moving data from mem to disk (a database term but now used commonly)
            * Reliability
                * SSD vs. HD vs. tape: SSD's fastest but least reliable. Tape's slowest but most
                  reliable.
    * CPU architectures
        * x86 and ARM: two different ISAs (Instruction Set Architectures)
        * CISC vs. RISC debate
        * 32-bit vs. 64-bit: basically register size but it has cascading effects.
            * With 32-bit, larger-size variable computations (e.g., 64-bit integer additions) need
              to be broken down to multiple operations. With 64-bit, those can be done with a single
              operation.
            * Register size ---> affects the pointer variable size
            * Pointer size ---> affects the memory address space size
            * Pointer size ---> also affects the memory access channel (bus) size
* Kernel
    * Kernel roles
        * Resource management: many programs want to access the hardware at the same time. The
          kernel manages (mediates) these accesses.
        * Program control: the kernel controls programs (running, stopping, etc.).
        * Protection: the kernel provides protection for users and programs. E.g., one user's data
          is not accessible by another user. One program's execution doesn't mess up with another
          program's execution.
    * Event-driven
        * The kernel is event-driven, i.e., it responds to an event.
        * An event can be a hardware interrupt (e.g., mouse click, keyboard press), program
          syscalls (e.g., printing out), and signals (e.g., SIGINT, SIGTERM).
    * User mode vs. kernel mode
            * Not to be confused with normal user vs. super user.
    * [The Linux kernel map](https://makelinux.github.io/kernel/map/)
        * Human interfaces: no term to introduce
        * System
            * Device drivers: every device needs a device driver that controls it. E.g., a network
              card device driver talks to the network card to send/receive data to/from the physical
              network.
        * Processing
            * Important terms are processes, threads, synchronization, and scheduling. Later
              lectures cover these.
        * Memory
            * Important terms are virtual memory, physical memory, and paging. Later lectures cover
              these.
        * Storage
            * Important terms are file systems and VFS (Virtual File System).
            * VFS is an interface. It defines data structures and operations that a file system
              should support, e.g., `read` and `write`. As long as a file system implements this
              interface, it can be plugged in easily. There are many file systems out there that are
              in use.
        * Networking
            * Important terms are sockets, TCP, UDP, and IP. Later lectures cover these.
* Applications
    * LLVM compilation exercise
        * Purpose: examining a large code base and compiling it.
        * The code structure is important, e.g., `include`, `lib`, `CMakeLists.txt`, etc., as
          everybody expects a similar code structure.
    * Lifetime of a program: code -> compilation -> executable -> memory loading -> execution
    * Two major ways to run a program
        * Compilation (C/C++ example)
        * Interpretation (Python example)
    * Main trade-off
        * Performance vs. portability
        * Performance is better for compilation since it directly generates machine code that you
          can execute. Interpretation has to translate high-level code to machine code at run time,
          so it's slower.
        * Compilation is not really portable because you compile for a specific ISA. You can't run
          the same executable on a different ISA. E.g., you can't copy an executable from an x86
          machine to an ARM machine and expect it to run. For interpretation, you can copy your
          script and run it on pretty much any machine.
    * Intermediate point: IR (Intermediate Representation)
        * Java bytecode, LLVM bitcode: these are architecture-neutral ISAs. They are low-level
          instructions similar to x86 or ARM instructions but they do not target specific CPUs.
        * Once you generate lower-level code based on an IR, you can use a backend compiler
          to compile it further down to an architecture-specific executable.
    * POSIX (Portable Operating System Interface)
        * A standard for software portability across different OSs.
        * Includes programming interface (file I/O, C standard library, etc.) and shell utilities
    * ABI (Application Binary Interface)
        * This is similar to API (Application Programming Interface) but it's at the binary level
          (after you compile your code). For an API, it is at the code level. I.e., you write your
          code according to the API and you can use a library that implements the API.
        * For ABI, it is an interface for a binary (an executable). If you want to run an executable
          for a certain OS, your compiler need to generate an executable that follows the ABI for
          the OS.
        * An OS defines an ABI to run a binary.
    * Coreutils: all the shell utilities that a POSIX-compliant system should have.
        * There are many implementations but the most popular one is [GNU
          Coreutils](https://www.gnu.org/software/coreutils).
    * Git exercise
        * Do the GitHub hello world exercise that does branching, pull requests, merging, etc.
        * Git pull.
