# Inter-Process Communication (IPC)

* Inter-process communication allows different processes (as well as threads) to communicate with
  each other.
* UNIX domain socket is an example of this, but there are other facilities---pipes, FIFOs, message
  queues, memory mapping, and shared memory.

## Pipes

* We already know shell pipes. For example,
    * `ps aux | grep bash`
    * `|` is a pipe.
    * The output of the first becomes input to the second.
* We can also use pipes programmatically.
    * `int pipe(int filedes[2])`
    * `man 7 pipe` gives us an overview in detail.
* The function returns two file descriptors.
    * `filedes[0]` gives us the read end.
    * `filedes[1]` gives us the write end.
* A pipe has the following characteristics.
    * It uses a buffer in the kernel.
    * It is unidirectional.
    * It is a byte stream.
* Since a pipe returns file descriptors, we can use regular file I/O functions from `stdio`.
    * We can use both non-buffered I/O (e.g., `read()`, `write()`) and buffered I/O (e.g.,
      `fprintf()`, `fscanf()`).
* A typical use case is parent-child communication.
    * First we create a pipe, and then we call `fork()` to create a child.
    * Both file descriptors (`filedes[0]` and `filedes[1]`) will be available for both the parent
      and the child. This is because memory is cloned.
    * After that, the parent and the child can communicate with each other with the pipe.
* An important point: typically each process uses a different end. In other words, each process
  closes the one that they don't use.
    * For example, if we want the child to write to the pipe and the parent to read from the pipe,
      we close the write end for the parent (`close(filedes[1])`) and we close the read end for
      the child (`close(filedes[0])`). Then, we write to the pipe for the child and read from the
      pipe for the parent.
    * Take a look at the example from `man pipe`.
* Another important point: `PIPE_BUF`
    * `man 7 pipe` has a `PIPE_BUF` section that describes this in detail.
    * If we call `write()` with less than `PIPE_BUF` bytes, it will be atomic.
    * If we call `write()` with more than `PIPE_BUF` bytes, it may be non-atomic.
    * "POSIX.1 requires `PIPE_BUF` to be at least 512 bytes.  (On Linux, `PIPE_BUF` is 4096 bytes.)"
    * However, the behavior depends on the blocking status. Again, the `PIPE_BUF` section of `man 7
      pipe` has the details.
* Yet another important point: closing the write end completely will trigger `read()` to return 0
  after returning everything in the buffer.
    * This can be used as a signaling mechanism.
    * An example scenario.
        * A parent `fork()`s and `read()`s.
        * Child processes all close the write end.
        * When the parent's `read()` returns everything in the buffer, it will return 0.
        * When the parent gets 0 from `read()`, it knows that all child processes have closed the
          write end.
* `int dup2(int oldfd, int newfd)`
    * If we want to redirect one program's standard output to another program's standard input
      (just as we would with a shell pipe, `|`), we can combine `dup2()` and `pipe()`.
    * The `dup2()` system call creates a copy of the file descriptor `oldfd` using the file
      descriptor number specified in `newfd`.
    * We can redirect all our program's standard output to the write end of the pipe.
        * If we call `dup2(filedes[1], STDOUT_FILENO)`, it duplicates `filedes[1]` to
          `STDOUT_FILENO`. Thus, if our program writes to `STDOUT_FILENO`, it will write to
          `filedes[1]`, i.e., the write end of the pipe.
    * We can redirect all our pipe's input to the standard input.
        * If we call `dup2(filedes[0], STDIN_FILENO)`, it duplicates `filedes[0]` to
          `STDIN_FILENO`. Thus, if our program reads from `STDIN_FILENO`, it will read from
          `filedes[0]`, i.e., the read end of the pipe.
* `FILE *popen(const char *command, const char *mode)`
    * `popen()` is a convenient function if we want to create a pipe to an existing program.
    * `command` is the program we want to execute and connect with a pipe.
    * `mode` determines read (`"r"`) from the command or write (`"w"`) to the command.
    * Use `pclose()` to close.
* Activity: modify the example in `man pipe` as follows.
    * The parent should send a string to the child.
    * The child should send the string back to the parent
    * The parent should print out the received string.
* One solution:

  ```bash
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/types.h>
  #include <sys/wait.h>
  #include <unistd.h>

  int main(int argc, char *argv[]) {
    int pipefd[2], pipefd2[2];
    pid_t cpid;
    char buf;

    if (argc != 2) {
      fprintf(stderr, "Usage: %s <string>\n", argv[0]);
      exit(EXIT_FAILURE);
    }

    if (pipe(pipefd) == -1) {
      perror("pipe");
      exit(EXIT_FAILURE);
    }

    if (pipe(pipefd2) == -1) {
      perror("pipe");
      exit(EXIT_FAILURE);
    }

    cpid = fork();
    if (cpid == -1) {
      perror("fork");
      exit(EXIT_FAILURE);
    }

    if (cpid == 0) {     /* Child reads from pipe and writes to another pipie */
      close(pipefd[1]);  /* Close unused write end */
      close(pipefd2[0]); /* Close unused read end */

      while (read(pipefd[0], &buf, 1) > 0)
        write(pipefd2[1], &buf, 1);

      write(pipefd2[1], "\n", 1);
      close(pipefd[0]);
      close(pipefd2[1]);
      _exit(EXIT_SUCCESS);

    } else { /* Parent writes argv[1] to pipe and reads from another pipe */
      close(pipefd[0]);  /* Close unused read end */
      close(pipefd2[1]); /* Close unused write end */
      write(pipefd[1], argv[1], strlen(argv[1]));
      close(pipefd[1]); /* Reader will see EOF */
      while (read(pipefd2[0], &buf, 1) > 0)
        write(STDOUT_FILENO, &buf, 1);
      close(pipefd2[0]);
      wait(NULL); /* Wait for child */
      exit(EXIT_SUCCESS);
    }
  }
  ```

## FIFOs

* Two or more related processes (parent, child, grandchild, etc.) can share a pipe as above.
  However, unrelated processes can't share a pipe.
* Instead, they can share a FIFO to communicate with each other.
* A FIFO is a *named pipe*.
    * `man 3 mkfifo` has the details.
    * `int mkfifo(const char *pathname, mode_t mode)`
    * `pathname` is the name of the FIFO to be created.
    * `mode` is the same as `open()`.
    * This is similar to UNIX domain sockets as it creates a file.
    * Use `unlink()` to remove a FIFO, just like a file.
* As long as a process knows `pathname`, it can access a FIFO. Thus, unrelated processes (assuming
  they all know `pathname`) can share a FIFO.
    * Once a FIFO file is created, different processes use `open()`, `read()`, `write()`, etc.
    * A FIFO is still unidirectional and it's typically for two processes.
    * One process should open it for read and the other should open it for write.
    * `open()` blocks until the other process calls `open()` as well.

## POSIX Message Queues

* A message queue is similar to a FIFO, but it is typically used to send a structured data (a
  *message*), e.g., `struct` or `union`, rather than a byte stream.
    * `man 7 mq_overview` details this.
* There are 5 important functions.
    * `mq_open()`, `mq_send()`, `mq_receive()`, `mq_close()`, and `mq_unlink()`
* `int mq_send(mqd_t mqdes, const char *msg_ptr, size_t msg_len, unsigned int msg_prio);`
    * A message queue sends a structured data using a pointer (`msg_ptr`) to the structured data.
    * The last parameter, `msg_prio` determines a priority of the message.
    * The queue is a priority queue, i.e., all the messages are ordered based on their priorities
      (and FIFO for the same priority).

## Memory Mapping

This is a slight detour for IPC but we will see the reason soon.

* `void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset)`
    * Creates what's known as a *memory mapping*.
    * `addr`: starting address of the new mapping. `NULL` is typical to let the OS pick the address
      for the new mapping.
    * `length`: the length of the mapping.
    * `prot`: executable, readable, writable, or not accessible.
    * `flags`: `MAP_SHARED`, `MAP_PRIVATE`, and `MAP_ANONYMOUS`. (Explained below.)
    * `fd`: file to be mapped. (Explained below.)
    * `offset`: the offset of the file to be mapped.
    * Returns a pointer to the beginning of the new mapping.
* There are two types of memory mappings.
    * *File* mapping
        * It maps a file to a memory region, allowing us to use a file as a memory region.
        * File I/O becomes memory access. Instead of `read()`/`write()` calls, we can just use
          pointers and variables to read from or write to a file.
        * This is called a *memory-mapped file*.
    * *Anonymous* mapping
        * Another way to allocate memory to our process (in addition to `sbrk()`). `malloc()` uses
          both `sbrk()` and `mmap()`.
* In addition, we can have *shared* or *private* memory mappings.
    * Multiple processes can share a mapping.
        * For example, we can create an file/anonymous mapping and `fork()` a child. Since memory
          is cloned, the parent and the child will share the same mapping.
        * Or, multiple processes can map the same file.
    * A process can create a private mapping.
        * In this case, each process will have a private copy.
* Thus, memory mapping has four cases.
    * Private file mapping: a file is mapped to a process as a private mapping. In other words, if
      multiple processes map the same file, they each will have a private copy. In other words, if
      they write to its mapping, it *won't* be written to the actual file.
    * Private anonymous mapping: more memory gets allocated to the calling process. `fork()` copies
      the memory but each process has a private copy. In other words, changes to the mapped memory
      are not propagated to the child (or vice versa).
    * Shared file mapping: a file is ampped to a process as a shared mapping. In other words, if
      multiple processes map the same file, changes will be propagated to all the processes and the
      actual file will be written.
    * Shared anonymous mapping: more memory gets allocated to the calling process. Moreover, memory
      is shared and changes are propagated. (More on this in [Shared Memory](#shared-memory).)
* `int munmap(void *addr, size_t length);`
    * Unmaps the mapped memory.
* Activity: memory-mapped file I/O.
    * Modify the example from `man mmap` as follows.
    * Receive only one command-line argument, which is a file name.
    * Create a file memory mapping for the entire file.
    * Print out the content of the entire memory mapping.
* One possible solution:

  ```bash
  #include <fcntl.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <sys/mman.h>
  #include <sys/stat.h>
  #include <unistd.h>

  #define handle_error(msg)                                                      \
    do {                                                                         \
      perror(msg);                                                               \
      exit(EXIT_FAILURE);                                                        \
    } while (0)

  int main(int argc, char *argv[]) {
    char *addr;
    int fd;
    struct stat sb;
    size_t length;
    ssize_t s;

    if (argc != 2) {
      fprintf(stderr, "%s file\n", argv[0]);
      exit(EXIT_FAILURE);
    }

    fd = open(argv[1], O_RDONLY);
    if (fd == -1)
      handle_error("open");

    if (fstat(fd, &sb) == -1) /* To obtain file size */
      handle_error("fstat");

    length = sb.st_size;

    addr = mmap(NULL, length, PROT_READ, MAP_PRIVATE, fd, 0);
    if (addr == MAP_FAILED)
      handle_error("mmap");

    s = write(STDOUT_FILENO, addr, length);
    if (s != length) {
      if (s == -1)
        handle_error("write");

      fprintf(stderr, "partial write");
      exit(EXIT_FAILURE);
    }

    munmap(addr, length);
    close(fd);

    exit(EXIT_SUCCESS);
  }
  ```

## Shared Memory

* Shared memory uses memory mapping across processes.
* `man 7 shm_overview` provides an overview.
    * We can first open a shared memory object (using `shm_open()` as below), truncate the size
      (using `ftruncate()` as below), and create a memory mapping with the shared memory object
      (using `mmap()` as below).
* `int shm_open(const char *name, int oflag, mode_t mode)`
    * `name` is assumed to be known and shared among processes. A general rule for naming is to use
      the form `/somename`.
    * Returns a file descriptor.
    * This function is similar to file open, but provides a file descriptor for a *shared memory
      object*.
        * This is just like creating a file. `/dev/shm/` is the directory for all shared memory
          object files, e.g. `ls /dev/shm/somename`.
    * `O_CREAT` flag determines whether we open a new object or an existing one.
    * `mode` is for permissions on creation.
    * Refer to `man shm_open` for the details.
* `int ftruncate(int fd, off_t length)`
    * When a memory object is created, the size is 0. We need to use `ftruncate()` to change the
      size.
    * Refer to `man ftruncate`.
* `void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset)`
    * Once we create a shared memory object and adjust the size, we need to create a memory
      mapping for it.
    * We need to give the shared memory object file descriptor as `fd`.
* `int munmap(void *addr, size_t length)`
    * When we no longer need to access the shared memory, we can unmap it.
* `int shm_unlink(const char *name)`
    * One we are done with the shared memory, we can remove the shared memory object.
    * This removes the shared memory object file from `/dev/shm/`.
    * However, if there are processes that still use the shared memory object, they can keep using
      it.
* Activity
    * Write two programs that communicates with each other via shared memory.
    * They should each receive a shared memory object file name as the only command-line argument.
    * One program should write an integer to the shared memory
    * The other program should read the integer written by the first program from the shared memory.
* One possible solution for the writer.

  ```bash
  #define _XOPEN_SOURCE 500

  #include <fcntl.h>
  #include <stdint.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/mman.h>
  #include <sys/stat.h>
  #include <sys/types.h>
  #include <unistd.h>

  int main(int argc, char *argv[]) {
    if (argc != 2) {
      fprintf(stderr, "Usage: %s /shm-path\n", argv[0]);
      exit(EXIT_FAILURE);
    }

    char *shmpath = argv[1];

    int fd = shm_open(shmpath, O_CREAT | O_RDWR, S_IRUSR | S_IWUSR);
    if (fd == -1) {
      perror("shm_open");
      exit(EXIT_FAILURE);
    }

    if (ftruncate(fd, sizeof(uint8_t)) == -1) {
      perror("ftruncate");
      exit(EXIT_FAILURE);
    }

    /* Map the object into the caller's address space */

    uint8_t *ptr =
        mmap(NULL, sizeof(*ptr), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    *ptr = 100;
  }
  ```

* One possible solution for the reader.

  ```bash
  #define _XOPEN_SOURCE 500

  #include <fcntl.h>
  #include <stdint.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/mman.h>
  #include <sys/stat.h>
  #include <sys/types.h>
  #include <unistd.h>

  int main(int argc, char *argv[]) {
    if (argc != 2) {
      fprintf(stderr, "Usage: %s /shm-path\n", argv[0]);
      exit(EXIT_FAILURE);
    }

    char *shmpath = argv[1];

    int fd = shm_open(shmpath, O_RDWR, S_IRUSR | S_IWUSR);
    if (fd == -1) {
      perror("shm_open");
      exit(EXIT_FAILURE);
    }

    if (ftruncate(fd, sizeof(uint8_t)) == -1) {
      perror("ftruncate");
      exit(EXIT_FAILURE);
    }

    uint8_t *ptr =
        mmap(NULL, sizeof(*ptr), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    printf("%d\n", *ptr);
  }
  ```
