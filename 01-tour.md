# A Tour of Computer Systems

* We're mainly discussing the *terminology* necessary for the course and generally for computer
  systems.
    * The terms that you will encounter in this course as well as any other contexts related to
      systems.
* The traditional OS stack: the hardware layer, the kernel layer, and the application layer

  ```bash
  +--------------+
  | applications |
  |--------------|  <-- syscall interface (the focus of the course)
  | kernel       |
  |--------------|
  | hardware     |
  +--------------+
  ```

* Systems programming is low-level programming that directly interacts with hardware or the OS,
  often using the syscall interface shown above.
    * Only a few low-level languages, e.g., C, C++, Rust, give you the ability to do systems
      programming, e.g., raw memory access. E.g., Python and Java don't allow you to do that.
* Higher-level programs don't typically need a systems programming language, unless it needs high
  performance.
* We need to choose a language that fits the target program's goals.

## The Hardware Layer

* (Look at [hardware-figures.pptx](hardware-figures.pptx) together for figures.)
* Hardware components: CPU, memory, and I/O devices.
* Two fundamental components in computing: computation and data
* This is captured in the current computer architecture in CPU and memory.

### The Evolution of CPU

* Moore's Law (look at [hardware-figures.pptx](hardware-figures.pptx))
* Originally, CPU was what is now known as single core, that could only run one sequence of
  instructions.
* The focus was speeding up single core until ~2000 (Moore's Law graph). Then around 2005, CPU
  designers shifted focus to adding more cores, i.e., running multiple sequences of code at the same
  time.

### The Evolution of Memory

* It doesn't matter how fast a CPU is since it has to go get the data from memory.
* CPU was getting faster, so memory access had to get faster as well.
* How the speed gap was addressed:
    * First, registers (very small memory inside a CPU) were added to load data items from memory.
      The distance from computation modules was and still is almost negligible, providing faster
      data access.
    * The chip technology started to get better and better, and CPU designers were able to pack more
      and more things within the same size area.
    * In addition to registers and computation units, they added cache. A cache is much larger in
      size than registers, but much smaller than memory. However, the distance from computation
      modules is still much smaller than memory.
    * Nowadays, there are more caches, e.g., L1 cache, L2 cache, and L3 cache. L1 is the smallest
      but closest to the CPU. L3 is the largest but the farthest from the CPU.

### Memory Hierarchy

* Registers, cache, mem, SSDs, hard drives, tapes.
    * Registers are the smallest in size but the closest to the CPU (fastest).
    * Tapes are the slowest but the largest in size.
* Trade-offs
    * Bigger size typically means more expensive (size correlates with price).
    * Distance (access speed): faster means closer to CPU.
    * Persistence
        * The term *commit* means moving data from memory to disk. More generally, it means changing
          the state of data from temporary to permanent. This was originally a database term but now
          used commonly, e.g., `git commit`.
    * Reliability
        * SSD vs. HDD vs. tape: SSD's fastest but least reliable. A tape is slowest but most
          reliable and lasts longer.

### CPU Architectures

* x86 and ARM: two different ISAs (Instruction Set Architectures) that define and use two different
  sets of instructions.
* 32-bit vs. 64-bit architectures
    * There are many aspects to this, but we only discuss what's relevant to 201.
    * The most relevant aspect for 201 is that this determines the register size. However, it has
      cascading effects.
    * With 32-bit, larger-size variable computations (e.g., 64-bit integer additions) need to be
      broken down to multiple operations. With 64-bit, those can be done with a single operation.
    * Register size ---> affects the pointer variable size (32-bit uses 32-bit pointers & 64-bit
      uses 64-bit pointers).
    * Pointer size ---> affects the memory address space size (32-bit space vs. 64-bit space).
    * Pointer size ---> also affects the memory access channel (bus) size.
        * A 64-bit address needs 64 bits to be transferred from the CPU to memory, while a 32-bit
          address needs 32 bits to be transferred. A 64-bit architecture typically has more wires (a
          wider bus) to transfer memory addresses.

## The Kernel Layer

### Kernel Roles

* Resource management: many programs want to access the hardware at the same time. The kernel
  manages (mediates) these accesses.
* Program control: the kernel controls programs (running, stopping, etc.).
* Protection: the kernel provides protection for users and programs. E.g., one user's data should
  not be accessible by another user. One program's execution shouldn't mess up with another
  program's execution.

### Event-Driven

* The kernel is event-driven, i.e., it responds to an event.
* An event can be a hardware *interrupt* (e.g., mouse click, keyboard press), a program *syscall*
  (e.g., printing out), and a *signal* (e.g., SIGINT, SIGTERM).

### User Mode vs. Kernel Mode

* Typically, an OS kernel runs in kernel mode. User applications run in user mode.
* A modern CPU can run in one of those two modes at a given moment.
* Kernel mode allows full privilege and full access to the hardware.
* User mode is more restricted than kernel mode. It cannot run certain instructions (e.g.,
  instructions that allow direct access to hardware). It also cannot access certain regions of
  memory set by the code that runs in kernel mode.
* This is different from normal user vs. super user. A super user is still a user, but with full
  administrative privileges. A super user can do things that a normal user isn't allowed to do and
  run programs that a normal user isn't allowed to run.

### The Linux Kernel Map

* [The Linux Kernel Map](https://makelinux.github.io/kernel/map/)
* Human interfaces: no term to introduce
* System
    * Device drivers: every device needs a device driver that controls it. E.g., a network card
      device driver talks to the network card to send/receive data to/from the physical network.
* Processing
    * Important terms are processes, threads, synchronization, and scheduling. Later lectures cover
      these.
* Memory
    * Important terms are virtual memory, physical memory, and paging. Later lectures cover these.
* Storage
    * Important terms are file systems and VFS (Virtual File System).
    * VFS is an interface. It defines data structures and operations that a file system should
      support, e.g., `read` and `write`. As long as a file system implements this interface, it can
      be plugged in easily. There are many file systems that are in use that implement VFS.
* Networking
    * Important terms are sockets, TCP, UDP, and IP. Later lectures cover these.

## The Application Layer

### Lifetime of a Program

* Code -> compilation -> executable -> memory loading -> execution
* Activity: LLVM compilation exercise
    * Purpose: examining a large code base and compiling it.
    * The code structure is important, e.g., `include`, `lib`, `CMakeLists.txt`, etc., as everybody
      expects a similar code structure.

### Compilation vs. Interpretation

* Two major ways to run a program
    * Compilation (e.g., C/C++)
    * Interpretation (e.g., Python, Bash)
* Main trade-off
    * Performance vs. portability
    * Performance is better for compilation since it directly generates machine code that you can
      execute. Interpretation has to translate high-level code to machine code at run time, so it's
      slower.
    * Compilation is not really portable because you compile for a specific ISA. You can't run the
      same executable on a different ISA. E.g., you can't copy an executable from an x86 machine to
      an ARM machine and expect it to run. For interpretation, you can copy your script and run it
      on pretty much any machine as long as there is an interpreter for it.
* Intermediate design point: IR (Intermediate Representation)
    * Java bytecode, LLVM bitcode: these are architecture-neutral ISAs. They are low-level
      instructions similar to x86 or ARM instructions but they do not target specific CPUs.
    * Once you generate lower-level code based on an IR, you can use a backend compiler to compile
      it further down to an architecture-specific executable.
    * Thus, an IR serves as a portable interface for higher-level languages. For example, many
      modern languages, such as Go, Rust, etc., all generate LLVM bitcode instead of ISA-specific
      machine code. They then use LLVM to generate machine code for a specific ISA.

### POSIX (Portable Operating System Interface)

* A standard for (user-level) software portability across different OSs.
* Includes programming interface (file I/O, C standard library, etc.) and shell utilities

### ABI (Application Binary Interface)

* This is similar to API (Application Programming Interface).
* An API is at the code level. I.e., you write your code according to the API and you can use a
  library that implements the API.
* An ABI is an interface for a binary (an executable) that an OS defines. If you want to run an
  executable on a certain OS, your compiler need to generate an executable that follows the ABI for
  the OS.
* For example, the Windows ABI is different from the Linux ABI. This is why you cannot copy a
  Windows binary (`.exe`) to a Linux machine and run it (and vice versa).

### Coreutils

* Coreutils are all the shell utilities that a POSIX-compliant system should have.
* There are many implementations but the most popular one is [GNU
  Coreutils](https://www.gnu.org/software/coreutils).
* Popular commands, such as `ls`, `cp`, etc., are all part of coreutils.

## How Virtualization Fits in the Traditional OS Stack

There are different types of virtualization.

### VMM (Virtual Machine Monitor) Directly atop Hardware

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

### VMM atop the Kernel

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
* This is often used in a desktop environment, e.g., VMWare Player, VirtualBox, QEMU.

### Containerization

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
* It combines many features that Linux provides for isolation. This includes process isolation
  (namespaces), resource control/isolation (cgroups), etc.
* This is the most popular form of virtualization these days, e.g., Docker, Podman.

### Activity: Git Exercise

* Do the [GitHub hello world
  exercise](https://docs.github.com/en/get-started/quickstart/hello-world) that does branching, pull
  requests, merging, etc.
* `git pull` is used to fetch from a remote repo. Read further from the [Atlassian's
  Tutorial](https://www.atlassian.com/git/tutorials/syncing/git-pull).
