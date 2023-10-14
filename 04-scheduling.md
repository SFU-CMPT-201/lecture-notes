# Scheduling Algorithms

* Scheduling
    * This is more about kernel internals but nevertheless an important topic, so one should know
      this even if one doesn't take an OS internals course.
    * Back in the day, there was only a single core but still there were many users and they all
      wanted to use the same system.
        * A question arises---how do you share the same CPU?
    * These days, there are many more processes you want to run than the # of cores.
* CPU scheduling
    * Sharing a core by multiple processes
    * Context switch: stop running one process and start running another process. This has overhead.

    ```bash
    ------> ready queue --------------> CPU -> terminate
        |                                |
        \--- I/O done <--- I/O queue <---/
    ```

    * Scheduling: Picking one from the ready queue
    * Types
        * Preemptive
            * The kernel stops a process any time.
        * Non-preemptive
            * The process gives up the core when it terminates or "waits" for something that takes a
              long time, e.g., I/O, child wait, sleep,
* Scheduling criteria
    * Maximize:
        * CPU utilization: keep the CPU as busy as possible
        * Throughput: # of processes that complete their execution per time unit
    * Minimize
        * Turnaround time: amount of time to execute a particular process (time from submission to
          termination)
        * Waiting time: amount of time a process has been waiting in ready queue
        * Response time: amount of time it takes from when a request is submitted until the first
          response is produced
* Scheduling algorithms
    * First Come, First Served
        * The straightforward one
        * Non-preemptive
    * Shortest Job First (SJF)
        * Assume for the sake of discussion, we know how long each process takes.
        * Non-preemptive
    * Shortest-Remaining-Time-First (SRTF)
        * Assume for the sake of discussion, we know how long each process takes.
        * Preemptive
        * This and SJF need to know how long processes take.
    * Side note: Interactive vs. batch
        * Interactive
            * Mainly user driven
            * Regular desktop applications
        * Batch
            * You run a program from start to end. No interaction in the middle.
            * Compiling a program
            * Data analytics
        * SJF & SRTF are mainly for batch processes (jobs).
    * Round Robin
        * Forget about knowing how long things take, let's just use round robin.
        * Preemptive
        * If the time unit is long, it's FCFS. If short, the context switch overhead is too much.
    * Priority
        * Motivation: real-time tasks with deadlines
        * Problem: starvation---lower priority processes may never run.
    * Multilevel Queue Scheduling
        * Solves starvation. Scheduling across queues, e.g., weighted RR: 1, 2, 3, 4, 5, 1, 2, 3, 4,
          1, 2, 3, 1, 2, 1, 1, 2, 3, 4, 5, ...
    * Multilevel Feedback Queue Scheduling
        * Like priority but based on CPU time
        * Strict priority across queues (Q0 should be empty then Q1).
        * Implements aging.
    * Linux Completely Fair Scheduler
        * Real-time processes
        * Normal processes
        * More time based on nice value
        * Virtual run time = physical run time + (decay formula)
            * Higher decay with lower priority
        * Sorted in red-black tree based on virtual run time
