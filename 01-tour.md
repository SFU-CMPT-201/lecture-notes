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
    * VMM (Virtual Machine Monitor) directly atop hardware

      ```bash
        VM1          VM2          VM3
      +--------+   +--------+   +--------+
      | Apps   |   | Apps   |   | Apps   |
      +--------+   +--------+   +--------+
      | Kernel |   | Kernel |   | Kernel |
      +--------+---+--------+---+--------|
      | VMM                              |
      |----------------------------------|
      | Hardware                         |
      +----------------------------------+
      ```

        * VMM emulates hardware for each VM (Virtual Machine).
        * This is often used in a data center environment.
    * VMM atop the kernel

      ```bash
        VM1          VM2
      +--------+   +--------+
      | Apps   |   | Apps   |
      +--------+   +--------+
      | Kernel |   | Kernel |
      +--------+---+--------+   +--------+
      | VMM                 |   | Apps   |
      +---------------------+---+--------+
      | Kernel                           |
      |----------------------------------|
      | Hardware                         |
      +----------------------------------+
      ```

        * A VMM is an application running atop a kernel, along with other applications.
        * The VMM creates/runs/manages VMs.
        * This is often used in a desktop environment, e.g., VMWare Player, VirutalBox, QEMU.
    * Containerization

      ```bash
        Container1          Container2
      +---------------+   +---------------+
      | Apps          |   | Apps          |
      +---------------+---+---------------+   +------+
      | Containerizer                     |   | apps |
      +-----------------------------------+---+------+
      | Kernel                                       |
      |----------------------------------------------|
      | Hardware                                     |
      +----------------------------------------------+
      ```

        * Containerization creates a *container* not a virtual machine.
        * It does not need to run another kernel.
        * It combines many features that Linux provides for isolation. This includes process
          isolation (namespaces), resource control/isolation (cgroups), etc.
        * This is the most popular form of virtualization these days, e.g., Docker, Podman.
* Hardware
    * (Look at [hardware-figures.pptx](hardware-figures.pptx) together for figures.)
    * Hardware components: CPU, memory, and I/O devices.
    * Two fundamental components in computing: computation and data
    * This is captured in the current computer architecture in CPU and memory.
    * The evolution of CPU
        * Moore's Law (look at [hardware-figures.pptx](hardware-figures.pptx))
        * Originally, CPU was what is now known as single core, that could only run one sequence of
          instructions.
        * The focus was speeding up single core until ~2000 (Moore's Law graph). Then around 2005,
          CPU designers shifted focus to adding more cores, i.e., running multiple sequences of code
          at the same time.
    * The evolution of memory
        * It doesn't matter how fast a CPU is since it has to go get the data from memory.
        * CPU was getting faster, so memory access had to get faster as well.
        * How the speed gap was addressed:
            * First, registers (very small memory inside a CPU) were added to load data items from
              memory. The distance from computation modules was and still is almost negligible,
              providing faster data access.
            * The chip technology started to get better and better, and CPU designers were able to
              pack more and more things within the same size area.
            * In addition to registers and computation units, they added cache. A cache is much
              larger in size than registers, but much smaller than memory. However, the distance
              from computation modules is still much smaller than memory.
            * Nowadays, there are more caches, e.g., L1 cache, L2 cache, and L3 cache. L1 is the
              smallest but closest to the CPU. L3 is the largest but the farthest from the CPU.
    * Hardware threading
        * A technique that makes a single core appear as multiple cores (threads).
    * NUMA (Non-Uniform Memory Access)
        * Look at [hardware-figures.pptx](hardware-figures.pptx).
        * One motherboard that has multiple CPUs (sockets).
        * Memory access time is non-uniform based on where the CPU is and where the memory is
          (farther away == takes more time to access).
    * Memory hierarchy
        * Registers, cache, mem, SSDs, hard drives, tapes.
            * Registers are the smallest in size but the closest to the CPU (fastest).
            * Tapes are the slowest but the largest in size.
        * Trade-offs
            * Bigger size typically means less expensive (size correlates with price).
            * Distance (access speed): faster means closer to CPU.
            * Persistence
                * The term *commit* means moving data from memory to disk. More generally, it means
                  changing the state of data from temporary to permanent. This was originally a
                  database term but now used commonly, e.g., `git commit`.
            * Reliability
                * SSD vs. HD vs. tape: SSD's fastest but least reliable. A tape is slowest but most
                  reliable and lasts longer.
    * CPU architectures
        * x86 and ARM: two different ISAs (Instruction Set Architectures) that define and use two
          different sets of instructions.
        * CISC vs. RISC debate
            * CISC (Complex Instruction Set Computer): CISC ISAs were mainly designed for (assembly)
              programmers and provided many convenient instructions to reduce the effort of
              (manually) writing (assembly) code.
            * RISC (Reduced Instruction Set Computer): RISC ISAs were mainly designed for compilers
              and provided a simplified set of instructions that a CPU can run efficiently.
            * Modern CPUs often combine CISC and RISC features.
        * 32-bit vs. 64-bit architectures
            * This is basically determined by the register size but it has cascading effects.
            * With 32-bit, larger-size variable computations (e.g., 64-bit integer additions) need
              to be broken down to multiple operations. With 64-bit, those can be done with a single
              operation.
            * Register size ---> affects the pointer variable size (32-bit uses 32-bit pointers &
              64-bit uses 64-bit pointers).
            * Pointer size ---> affects the memory address space size (32-bit space vs. 64-bit
              space).
            * Pointer size ---> also affects the memory access channel (bus) size.
                * A 64-bit address requires 64 bits to be transferred from the CPU to memory, while
                  a 32-bit address requires 32 bits to be transferred. I.e., A 64-bit architecture
                  requires more wires to transfer memory addresses.
* Kernel
    * Kernel roles
        * Resource management: many programs want to access the hardware at the same time. The
          kernel manages (mediates) these accesses.
        * Program control: the kernel controls programs (running, stopping, etc.).
        * Protection: the kernel provides protection for users and programs. E.g., one user's data
          should not be accessible by another user. One program's execution shouldn't mess up with
          another program's execution.
    * Event-driven
        * The kernel is event-driven, i.e., it responds to an event.
        * An event can be a hardware *interrupt* (e.g., mouse click, keyboard press), program
          *syscalls* (e.g., printing out), and *signals* (e.g., SIGINT, SIGTERM).
    * User mode vs. kernel mode
            * Typically, an OS kernel runs in kernel mode. User applications run in user mode.
            * A modern CPU can run in one of those two modes at a given moment.
            * Kernel mode allows full privilege and full access of the hardware.
            * User mode is more restricted than kernel mode. It cannot run certain instructions
              (e.g., instructions that allow direct access of hardware). It also cannot access
              certain regions of memory set by the code that runs in kernel mode.
            * This is different from normal user vs. super user. A super user is still a user (with
              administrative permissions). A super user can do things that a normal user aren't
              allowed to do and run programs that a normal user aren't allowed to run.
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
              interface, it can be plugged in easily. There are many file systems that are in use
              that implement VFS.
        * Networking
            * Important terms are sockets, TCP, UDP, and IP. Later lectures cover these.
* Applications
    * Activity: LLVM compilation exercise
        * Purpose: examining a large code base and compiling it.
        * The code structure is important, e.g., `include`, `lib`, `CMakeLists.txt`, etc., as
          everybody expects a similar code structure.
    * Lifetime of a program: code -> compilation -> executable -> memory loading -> execution
    * Two major ways to run a program
        * Compilation (e.g., C/C++)
        * Interpretation (e.g., Python, Bash)
    * Main trade-off
        * Performance vs. portability
        * Performance is better for compilation since it directly generates machine code that you
          can execute. Interpretation has to translate high-level code to machine code at run time,
          so it's slower.
        * Compilation is not really portable because you compile for a specific ISA. You can't run
          the same executable on a different ISA. E.g., you can't copy an executable from an x86
          machine to an ARM machine and expect it to run. For interpretation, you can copy your
          script and run it on pretty much any machine as long as there is an interpreter for it.
    * Intermediate design point: IR (Intermediate Representation)
        * Java bytecode, LLVM bitcode: these are architecture-neutral ISAs. They are low-level
          instructions similar to x86 or ARM instructions but they do not target specific CPUs.
        * Once you generate lower-level code based on an IR, you can use a backend compiler to
          compile it further down to an architecture-specific executable.
        * Thus, an IR serves as a portable interface for higher-level languages. For example, many
          modern languages, such as Go, Rust, etc., all generate LLVM bitcode instead of
          ISA-specific machine code. They then use LLVM to generate machine code for a specific ISA.
    * POSIX (Portable Operating System Interface)
        * A standard for (user-level) software portability across different OSs.
        * Includes programming interface (file I/O, C standard library, etc.) and shell utilities
    * ABI (Application Binary Interface)
        * This is similar to API (Application Programming Interface).
        * An API is at the code level. I.e., you write your code according to the API and you can
          use a library that implements the API.
        * An ABI is an interface for a binary (an executable) that an OS defines. If you want to run
          an executable on a certain OS, your compiler need to generate an executable that follows
          the ABI for the OS.
        * For example, the Windows ABI is different from the Linux ABI. This is why you cannot copy
          a Windows binary (`.exe`) to a Linux machine and run it (and vice versa).
    * Coreutils
        * Coreutils are all the shell utilities that a POSIX-compliant system should have.
        * There are many implementations but the most popular one is [GNU
          Coreutils](https://www.gnu.org/software/coreutils).
        * Popular commands, such as `ls`, `cp`, etc., are all part of coreutils.
    * Activity: Git exercise
        * Do the [GitHub hello world
          exercise](https://docs.github.com/en/get-started/quickstart/hello-world) that does
          branching, pull requests, merging, etc.
        * `git pull` is used to fetch from a remote repo. Read further from the [Atlassian's
          Tutorial][(https://www.atlassian.com/git/tutorials/syncing/git-pull).
