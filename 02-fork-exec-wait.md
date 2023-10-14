# Fork, Exec, and Wait

* How to read a man page
    * Read the function signature first (and the header files). Carefully examine the return type
      and all parameters. A parameter can be either an input or an output. Find out what it is for
      each parameter.
    * Check if there are Feature Test Macro Requirements. If there are, make sure you follow them
      correctly.
    * Read the description section, but often you need to quickly figure out which part of the
      description is relevant to what you try to do.
    * Read the return value section. Check if errors are returned and what they are.
    * Read the error section to find out more about what errors there are.
* `fork()` activity
    * Write a program that keeps calling `sleep()` with some timeout value.
        * Run it and check out `btop`.
            * Use the tree view. Check the PIDs.
            * Every process has a parent except for two (`init` and `kthreadd`, which are directly
              created by the kernel at boot time).
        * `a.out` should a child of the shell (`zsh`).
    * Read the man page for `fork()`.
    * Write a program that calls `fork()` and then keeps calling `sleep()` with some timeout value.
        * Run it and check out `btop` in tree mode. Kill them.
    * Write a program that calls `fork()` and prints out "parent" and the child PID if it is the
      parent process. If it is the child, print out "child" as well as its PID and the parent's PID.
      `man getpid` and `man getppid` to find out how to get the necessary PIDs.
        * The parent and the child need to do different things. Use `if-else` with the return value
          of `fork()` to differentiate the behavior.
* `exec()` activity
    * `man exec`
    * Write a program that creates a child process. The parent should call any one of `exec`
      functions that executes `ls -al`. The child should execute `exa -al`.
        * This actually needs to be `ls -a -l`.
* `wait()` activity
    * `man wait`
    * write a program that first creates a child process. The child process should run `exa -al`.
      The parent process should wait for the child process to terminate using `waitpid()`. If the
      child exited normally, print out "Child done." If not, print out the exit status.
* Zombie and orphan processes
    * Zombie
        * The child process terminates but the parent process hasn't called `wait()` yet.
        * This does not get cleaned up and occupies memory.
    * Orphan
        * The child process is running but the parent process has terminated.
        * On Linux, the child process becomes a child process of `init`.
* `errno`.
    * `man errno`
    * Write a fork bomb. Need to get the `errno` and print out corresponding definition (`EAGAIN`,
      `ENOMEM`, etc.). Also use `perror()` to print out the corresponding system error message.
