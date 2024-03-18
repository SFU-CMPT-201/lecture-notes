# File I/O

There are three things to know before we look at the actual system calls.

## Basic System Calls

There are five basic system calls for file I/O.

* `open()`
    * Read `man open` for a detailed description.
    * `open()` receives either 2 or 3 parameters.
        * `int open(const char *pathname, int flags);`
        * `int open(const char *pathname, int flags, mode_t mode);`
    * The 2nd parameter (`int flags`) specifies an access mode and a creation mode.
        * From `man open`, "The argument flags must include one of the following access modes:
          `O_RDONLY`, `O_WRONLY`, or `O_RDWR`. These request opening the file read-only, write-only,
          or read/write, respectively. In addition, zero or more file creation flags and file status
          flags can be bitwise-or'd in flags. The file creation flags are `O_CLOEXEC`, `O_CREAT`,
          `O_DIRECTORY`, `O_EXCL`, `O_NOCTTY`, `O_NOFOLLOW`, `O_TMPFILE`, and `O_TRUNC`."
    * The 3rd parameter
        * If creation is specified (either `O_CREAT` or `O_TMPFILE`), this specifies the file
          permissions.
    * `open()` returns a file descriptor.
    * What is a file descriptor?
        * A file descriptor is a handle for the file for your program to read and write.
        * A file descriptor is a small non-negative integer (`int`).
        * It could change every time you open the file.
        * When using a file, we need to *open* it first with `open()`.
    * *File offset*
        * A file opened for read and write is associated with the file offset, which is a pointer to
          a specific byte within a file. This is basically the position where `read()` or `write()`
          occurs.
        * `read()` automatically increments this file offset as it reads new bytes from the file as
          described below. This is so that subsequent calls to `read()` can continue reading from
          the last byte `read()` read. `write()` does the same as described below.
        * We can use `lseek()` described below to manually adjust the file offset as well.
* `write()`
    * Read `man 2 write` for a detailed description.
    * `ssize_t write(int fd, const void *buf, size_t count);`
    * `write()` writes to a file descriptor and *returns the number of bytes written*.
    * One important point is, from `man 2 write`, "The number of bytes written may be less than
      `count` if, for example, there is insufficient space on the underlying physical medium, or the
      `RLIMIT_FSIZE` resource limit is encountered (see setrlimit(2)), or the call was interrupted
      by a signal handler after having written less than `count` bytes."
    * Another important point is, from `man 2 write`, "For  a seekable file (i.e., one to which
      lseek(2) may be applied, for example, a regular file) writing takes place at the file off‐
      set, and the file offset is incremented by the number of bytes actually written."
* `read()`
    * Read `man 2 read` for a detailed description.
    * `ssize_t read(int fd, void *buf, size_t count);`
    * `read()` reads from a file descriptor and *returns the number of bytes read*.
    * One important point is, from `man 2 read`, "On files that support seeking, the read operation
      commences at the file offset, and the file offset is incremented by the number of bytes read.
      If the file offset is at or past the end of file, no bytes are read, and read() returns zero."
    * Another important point is, from `man 2 read`, "It is not an error if (the return value) is
      smaller than the number of bytes requested; this may happen for example because fewer bytes
      are actually available right now (maybe because we were close to end-of-file, or because we
      are reading from a pipe, or from a terminal), or because read() was interrupted by a signal."
* `close()`
    * `close()` closes the file descriptor.
* `lseek()`
    * Read `man lseek` for a detailed description.
    * We can use `lseek` to manually adjust the file offset.
    * `off_t lseek(int fd, off_t offset, int whence);`
    * The last parameter `whence` indicates from which location we want to adjust the file offset.
      There are three possibilities (`SEEK_SET`, `SEEK_CUR`, and `SEEK_END`) and they are
      illustrated below.

  ```bash
  Suppose a file has N bytes (i.e., EOF is N-1) and the current file offset is C.
  | 0 | 1 | ... | C | ... | N-2 | N-1 | N | ....

  0: SEEK_SET (Beginning of the file)
  C: SEEK_CUR (Current file offset)
  N: SEEK_END (End of the file)
  ```

* `fcntl()`
    * Read `man fcntl` for a detailed description.
    * `fcntl()` is used for file control.
    * It can do many things, but one use is to modify the flags and mode used when opening a file
      with `F_SETFL`. For example, we can open a file as read-only and later change it to read-write
      with `fcntl()` and `F_SETFL`.
* These are all system calls.
* Activity: write a program that writes X bytes to a file, move the file offset backward by X/2
  bytes, and read and print out from the offset to EOF.

  ```c
  #include <fcntl.h>
  #include <string.h>
  #include <sys/stat.h>
  #include <unistd.h>

  int main() {
    char *str = "hello world";
    char buf[12];
    // open file for reading and writing with read and write
    // permission for user; create if it doesn't exist.
    int fd = open("tmp", O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
    write(fd, str, strlen(str));
    lseek(fd, -6, SEEK_CUR);
    read(fd, buf, 6);
    write(STDOUT_FILENO, buf, 6);
    // zsh shows % to indicate that there's no terminating new line.
    close(fd);
  }
  ```

## Buffered I/O: `write()` vs. `fprintf()` vs. `printf()` (or `read()` vs. `fscanf()` vs. `scanf()`)

* The discussion here applies to I/O functions in three categories.
    * All I/O functions that start with `f`: `fprintf()`, `fscanf()`, `fputs()`, `fgets()`,
      `fput()`, `fget()`, etc.
    * The same functions without `f`: `printf()`, `scanf()`, `puts()`, `gets()`, etc.
    * I/O functions that are system calls: `write()`, `read()`, etc.
* For the simplicity of discussion, we will use `write()`, `fprintf()`, and `printf()` as examples
  here.
* `write()` is a system call, while `fprintf()` and `printf()` are standard library (stdio)
  functions.
* `write()` vs `fprintf()`
    * `write()` directly sends data to the kernel, while `fprintf()` manages a buffer in memory and
      writes to the buffer. They use `write()` under the hood. Because of this, `fprintf()` is
      called *buffered I/O*.
    * Again, this applies to `read()` vs. `fscanf()` as well. `read()` is a system call and
      `fscanf()` is a buffered I/O call.
    * The reason why you want to do this is due to the overhead associated with system calls. A
      system call needs to cross the boundary between user space and kernel space, which incurs
      significant overhead. Less syscalls, better performance.
    * `write()` takes a *file descriptor* (`int`) to write.
        * `ssize_t write(int fd, const void *buf, size_t count);`
    * `fprintf()` takes a *stream* (`FILE *`) to write.
        * `int fprintf(FILE *stream, const char *format, ...);`
* What is a stream?
    * Convenient wrapper around a file descriptor. Used by the stdio functions.
    * You can think of this as a file descriptor plus a buffer backing it up.
* Relationship

  ```bash
    user data
        |
        |
  stdio library calls (fprintf(), fscanf(), etc.)
        |
        |
  stdio buffer (memory)
        |
        |
  I/O system calls (write(), read(), etc.)
        |
    kernel
  ```

    * For example, suppose a user program has user data to write. If it uses `fprintf()`, it writes
      the data to the associated buffer in memory. At some point later, the data in the buffer will
      be passed to the kernel via `write()`. Then the kernel will write to the actual disk.
    * You can get the file stream from a file descriptor with `fdopen()`. You can get the file
      descriptor from a file stream with `fileno()`.
* Activity
    * Write a program that `open()`s a file `tmp`, `write()`s a string to `tmp`, and goes into an
      infinite loop that `sleep()`s for 30 seconds for each iteration. Run it in the background and
      see if it writes to `tmp`. (It should.) Delete `tmp`.

      ```c
      #include <fcntl.h>
      #include <string.h>
      #include <sys/stat.h>
      #include <unistd.h>

      int main() {
        char *str = "hello\n";
        int fd = open("tmp", O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
        write(fd, str, strlen(str));
        while (1) {
          sleep(30);
        }
      }
      ```

    * Write another program that opens a file `tmp` using `fopen()`, write a string to it using
      `fprintf()`, and goes into an infinite loop that `sleep()`s for 30 seconds for each iteration.
      Run it in the background and see if it writes to `tmp`. (It shouldn't.) Delete `tmp`.

      ```c
      #include <stdio.h>
      #include <unistd.h>

      int main() {
        char *str = "hello\n";
        FILE *f = fopen("tmp", "w+");
        fprintf(f, "%s", str);
        while (1) {
          sleep(30);
        }
      }
      ```

* `fflush()` immediately sends the buffered data to the kernel.
    * Calling `setbuf()` with `NULL` as the buffer automatically does flushing. Read `man setbuf`
      for more details.
* Activity: change the above program with `fprintf()` to add `fflush()`. Run it and see if it writes
  to `tmp`. (It should.)

  ```c
  #include <stdio.h>
  #include <unistd.h>

  int main() {
    char *str = "hello\n";
    FILE *f = fopen("tmp", "w+");
    fprintf(f, "%s", str);
    fflush(f);
    while (1) {
      sleep(30);
    }
  }
  ```

## Kernel Buffering

* The kernel has read/write buffers too. Even if we send data to the kernel, it doesn't immediately
  write to the disk.
* We can force this with `fsync()`.
* Relationship

  ```bash
    user data
        |
        |
  stdio library calls (fprintf(), fscanf(), etc.)
        |
        |
  stdio buffer (memory)
        |
        |
  I/O system calls (write(), read(), etc.)
        |
        |
    kernel
        |
        |
  kernel buffer (memory)
        |
        |
      disk
  ```

* Using `O_SYNC` when with `open()` automatically does `fsync()`.
* Parallel between user buffering and kernel buffering
    * `fflush()` and `fsync()`: both flush their buffer.
    * `setbuf()` with a `NULL` buffer and `O_SYNC`: both automatically perform no buffering.

## Blocking vs Nonblocking I/O

* A blocking call doesn't return until the operation can be done. For example, a blocking `read()`
  call doesn't return until there's something to read.
* A non-blocking call: `O_NONBLOCK` flag (either with `open()` or with `fcntl()` & `F_SETFL`)
    * If an operation can't be done immediately, the call returns an error, typically `EAGAIN`.

## The Universality of I/O

* The UNIX I/O model is often referred to as the "everything is a file" model. A file is used not
  only for actual files on stored on disks but also for all I/O devices such as the terminal, the
  network, etc.
* This means that even for a device (e.g., a terminal, a keyboard, a network interface card), there
  is a "file" associated with it, and we can use the same system calls (`open()`, `read()`,
  `write()`, `close()`, etc.) to interact with it. This is the universality of I/O. E.g., getting a
  key press from a keyboard is getting an input from the file associated with the keyboard. Sending
  data to a network is writing to the file associated with the network.
* Example: The `/proc` file system
    * It shows system and process information.
    * The kernel dynamically populates the information in the form of files. But they are not real
      files stored on disks.
    * `/proc/cpuinfo` shows the CPU info.
    * `/proc/meminfo` shows the memory info.
    * `/proc/PID/status` shows the process info.
    * `/proc/PID/fd` shows the file descriptor info.
    * `/proc/PID/task/TID` shows the thread info.
* Example: Terminal
    * There are three standard file descriptors that are always open.
    * (These are actually opened by the init process. `fork()` clones opened file descriptors
      because those are in memory. Thus, child processes also have them.)

      ```bash
      File descriptor | Purpose         | POSIX name    | stdio stream
      ----------------------------------------------------------------
      0               | standard input  | STDIN_FILENO  | stdin
      1               | standard output | STDOUT_FILENO | stdout
      2               | standard error  | STDERR_FILENO | stderr
      ```

* Example: Device files
    * On Linux, a device corresponds to a device file.
    * `/dev/` shows them.
    * Some of them are real devices, e.g., a mouse, a disk. Others are virtual devices.
        * `/dev/null` provides a "black hole" of all data written to it.
        * `/dev/zero` provides infinite null characters.
        * `/dev/random` and `/dev/urandom` are pseudorandom number generators.
* Example: The `/sys` file system
    * It shows kernel-internal information, e.g., various device setups, kernel subsystem info, etc.
* `ioctl`
    * I/O for things outside of the universal I/O model.

## Disk Partitions

* A disk is divided into partitions.
    * `/proc/partitions` shows the partition info.
* A partition is typically used as a file system.
    * A file system is a system that manages files and directories. There are different ways of
      doing it, and there are many file systems in use.
    * You can use different file systems on different partitions.
    * From the user's perspective, you can think of a file system as a file *tree* that starts with
      the root directory `/`.
    * In this sense, you can think of a partition as a storage entity that contains a file tree.
    * See below [Mounting and Unmounting](#mounting-and-unmounting) for a further discussion on file
      trees.
* A partition is also used as a swap space for memory management (e.g., paging).
    * `/proc/swaps` shows the swap space info.
    * However, you don't always need to have swap space.

## I-Nodes

* A file is associated with an i-node.
    * An i-node contains metadata about the file, e.g., file type, permissions, owner, timestamps,
      etc.
    * An i-node is identified by a number. `ls -li` shows i-node numbers.
* `stat()`, `lstat()`, and `fstat()`
    * These functions show file metadata mostly from the i-node.
    * Read `man 2 stat` for more details.
* Activity: use `stat()` to retrieve and display the file type of a regular file and a directory.
  Read `man inode`, especially about `st_mode`.

  ```c
  #include <stdio.h>
  #include <sys/stat.h>

  int main(int argc, char *argv[]) {
    char *path = argv[1];
    struct stat st;

    stat(path, &st);
    if (S_ISREG(st.st_mode)) {
      printf("Regular file\n");
    } else if (S_ISDIR(st.st_mode)) {
      printf("Directory\n");
    }
  }
  ```

## Hard/Soft Links

* Hard links: we can give many names to the same file.
    * A hard link is giving a name to a file.
* Activity: use `ln` to create a hard link to a file. Read `man ln` to figure out how to create a
  hard link.
    * Run `ls -li` for both the original file and the hard link. They're exactly the same.
        * `ls -li` shows the number of links as well (the third column).
        * The number should increase as more hard links are created.
    * Modify the content of the original and check the content of the hard link (and vice versa).
        * They should be the same.
    * `rm` only deletes the hard link.
        * `rm` is actually `unlink`.
        * Similarly, there's a system call used for deleting a file, which is `unlink()`. (There's
          also a more convenient one, `remove()`).
        * Only when there's no link left any more, the file gets deleted.
* Hard link limitations
    * No hard link is allowed for a directory. This is to prevent circular links, i.e., a child
      directory that links to the parent directory.
    * Hard links should be within the same file system. This is because a hard link is giving
      another name to an existing file.
* Soft links: also called "symbolic" link.
    * Unlike a hard link, a soft link (a.k.a. a symbolic link or a sym link) is an actual file. The
      content of the file is the path to the original file.
    * There's a system call `symlink()`.
* Activity: create a sym link with `ln -s`.
    * Try `ls -li` for both. They each have a unique i-node number, meaning they are two different
      files.
    * The hard link count does not change even if you create a sym link. Again, it's because it's a
      different file.
    * The sym link will point to nothing if the original gets deleted.
        * This is called a *dangling* link.
* No limitations like hard links
    * Sym links are allowed for directories.
    * Sym links do not have to be within the same file system.

# Setuid Bit, Setgid Bit, and Sticky Bit

* Setuid bit: if this is set, the user that runs the program can act as the owner of the program.
    * `passwd` example: `passwd`'s owner is the root user and the setuid bit is set. While running
      this program, a user can temporarily become the root. This is necessary to update the password
      file (e.g., `/etc/shadow` on Linux).
* Setgid bit: if this is set, the user that runs the program can act as if the user belonged to the
  group of the program.
* Sticky bit: if this is set on a directory, a user can delete or rename a file under it only when
  the user both has the write permission on the directory and owns the file or the directory. For
  example:
    * Suppose we create a `test` directory that is write-open for others (e.g., `rw-rw-rw-`).
    * Suppose a user `steveyko` creates a file in it.
    * Another user `cmpt201` can delete it.
    * If we add a sticky bit to `test` (using `chmod +t`). User `cmpt201` cannot delete the
      files/directories created by `steveyko`.
    * This is useful for sharing a directory for many users.

## VFS

* VFS (Virtual File System) defines an interface that different file systems can implement.
* `open`, `read`, `write`, `close`, etc. have all corresponding operations in the kernel, and VFS
  defines those operations.
* If a file system implements this interface, it can be used as a Linux file system.

## Mounting and Unmounting

* When you look at your Linux installation, there's a single file tree, which starts with the root
  directory `/`.
* In reality, this single file tree is actually multiple file trees combined together.
    * As discussed earlier, a partition contains a file tree, and there can be multiple partitions
      on a single disk. Also, there can be multiple disks for a single machine. Thus, there are
      typically multiple file trees on a single machine.
* We can "stitch" together different file trees from different partitions or disks and construct a
  single file tree.
* This is called *mounting*.
    * All file systems (from different partitions/disks) are mounted and form a single file tree.
* The `mount` command mounts a file tree (a file system) to a specific directory, and this is called
  the mount point.
    * The `mount` command also shows the current setup. This shows the same information as
      `/proc/mounts`.
* The `umount` command unmounts a file system.
