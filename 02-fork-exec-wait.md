# `fork()`, `exec()`, `wait()`, and `errno`

## How to Read a Man Page

* Reading a man page is the main way to learn how to use the functions/system calls for systems
  programming. However, it takes practice to effectively read a man page.
* The command is `man` followed by the thing that you want to read about, e.g., `man ls`, `man cd`.
  A lot of times, this will give you the right page.
* However, sometimes you need to specify a section in order to get the right page. The following
  three sections are most relevant to 201.
    * `man 1`: General commands, e.g., `man 1 ls`
    * `man 2`: System calls, e.g., `man 2 fork`
    * `man 3`: C standard library functions, e.g., `man 3 printf`
* Read the function signature first (and the header files). Carefully examine the return type and
  all parameters. A parameter can be either an input or an output. Find out what it is for each
  parameter.
* Check if there are Feature Test Macro Requirements. If there are, make sure you follow them
  correctly.
* Read the description section, but often you need to quickly figure out which part of the
  description is relevant to what you try to do.
* Read the return value section. Check if errors are returned and what they are.
* Read the error section to find out more about what errors there are.

## Process

* What is a process?
    * The simplest way to define a process is a *running program*.
    * However, it is not quite as simple as that, and we'll learn about it more.
* Program in memory
    * When you run a program, the OS creates a memory space for the program to run.
    * You will learn what this memory space contains in a later assignment.
    * One thing that is relevant to this lecture is the code of the program. It gets loaded to the
      memory space for the program so that the CPU can easily access the code and run it.
* Programmers can use system calls to control processes.
    * Three basic system calls: `fork()`, `exec()`, and `wait()`.

## Activity: `fork()`

* `fork()` creates a child process that is an identical copy of the calling process.
* Write a program that keeps calling `sleep()` with some timeout value.
    * Run it and check out `btop`.
        * Use the tree view. Check the PIDs.
        * Every process has a parent except for two (`init` and `kthreadd`, which are directly
          created by the kernel at boot time).
    * `a.out` should be a child of the shell (`zsh`).
* Read the man page for `fork()`.
* Write a program that (just) calls `fork()` and then keeps calling `sleep()` with some timeout
  value.
    * Run it and check out `btop` in tree mode. There should be a new child process.
    * Kill both processes.
* Write a program that calls `fork()` and prints out "parent" and the child PID if it is the parent
  process. If it is the child, print out "child" as well as its PID and the parent's PID. Use `man
  getpid` and `man getppid` to find out how to get the necessary PIDs.
    * The parent and the child need to do different things. Use `if-else` with the return value of
      `fork()` to differentiate the behavior.
* It is important to know that up to `fork()` there is a single process. After `fork()`, there are
  two copies of the same program running concurrently.

## Activity: `exec()`

* `man 3 exec`
    * Pay attention to what the man page says about `arg0`.
* An `exec()` function executes a new program. When it is called, the calling process removes the
  currently-running program from memory, loads the new program into memory, and starts executing the
  new program. This means that the calling process is completely replaced with the new program.
* What this means is that after calling `exec()`, the rest of the program (i.e., its code) won't get
  executed as it's gone from memory.
* Write a program that creates a child process. The parent should call any one of `exec` functions
  that executes `ls -a`. The child should call any one of `exec` functions that executes `ls -a -l
  -h` (which is the same as `ls -alh` but spelled out, which is necessary for `exec` functions).
    * Which process(es) will execute the print-out line? Parent? Child? Both? Neither?

## Activity: `wait()`

* `man 3 wait`
* A `wait()` function waits on a child process's termination and obtains its status.
* Write a program that first creates a child process. The child process should run `exa -al`. The
  parent process should wait for the child process to terminate using `waitpid()`. If the child
  exited normally, print out "Child done." If not, print out the exit status.
* Example code

  ```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <unistd.h>
  #include <wait.h>

  int main() {
    pid_t pid = fork();

    if (pid) {
      int wstatus = 0;
      if (waitpid(pid, &wstatus, 0) == -1) {
        perror("waitpid");
        exit(EXIT_FAILURE);
      }

      if (WIFEXITED(wstatus)) {
        printf("Child done.\n");
      } else {
        printf("%d\n", WEXITSTATUS(wstatus));
      }
    } else {
      if (execl("/usr/bin/exa", "-a", "-l", NULL) == -1) {
        perror("execl");
        exit(EXIT_FAILURE);
      }
    }

    return 0;
  }
  ```

* Zombie and orphan processes
    * Zombie
        * This is the state where the child process terminates but the parent process hasn't called
          `wait()` yet.
        * When it happens, the kernel does not clean up the memory for the child process entirely
          and keeps certain pieces of information. The reason is that the parent process, at some
          later point, might want to know the exit status of the child process and call `wait()`.
        * If the child process is in this state, it is called a zombie process (i.e., the process's
          dead but not completely).
        * If the parent creates many child processes and doesn't call `wait()` on them for a long
          time, there will be many zombie processes occupying memory, which can be problematic. For
          example, the kernel may not be able to create new processes anymore. Thus, it is important
          to call `wait()` on child processes in a timely manner.
        * On Linux, if the parent terminates without calling `wait()` on the child process, it is
          adopted by `init` and `init` calls `wait()` on the child process.
    * Orphan
        * This is the state where the child process is running but the parent process has
          terminated.
        * This process is called an orphan process as it doesn't have the parent process anymore.
        * On Linux, the child process becomes a child process of `init`.

## Activity: `errno`

* `man errno`
* `errno` is defined in `errno.h` and set by system calls and library functions when there is an
  error. It is used to further explain which error has occurred.
* Write a fork bomb. It needs to get the `errno` and print out corresponding definition
  (`EAGAIN`, `ENOMEM`, etc.). Also use `perror()` to print out the corresponding system error
  message.
* Example code

  ```c
  #include <errno.h>
  #include <stdio.h>
  #include <unistd.h>

  int main() {
    while (1) {
      if (fork() == -1) {
        char *str = NULL;
        switch (errno) {
        case EAGAIN:
          str = "EAGAIN";
          break;
        case ENOMEM:
          str = "ENOMEM";
          break;
        case ENOSYS:
          str = "ENOSYS";
          break;
        default:
          break;
        }
        perror("fork");
        printf("%s\n", str);
      }
    }
  }
  ```
