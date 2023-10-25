# Scheduling Algorithms

* This is more about kernel internals but nevertheless an important topic, so one should know this
  even if one doesn't take an OS internals course.
* Back in the day, there was only a single core but still there were many users and they all wanted
  to use the same system.
    * A question arises---how do you share the same CPU?
    * These days, there are many more processes you want to run than the # of cores. Thus, the same
      question remains.
* You can read more about this topic in the following chapters of
  [OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/). Keep in mind that OSTEP's discussions are much
  more in depth, which we consider out of scope for this course. Thus, you can use it as a reference
  but the main source should still be this lecture note.
    * [Chapter 7 Scheduling: Introduction](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched.pdf)
    * [Chapter 8 Scheduling: The Multi-Level Feedback
      Queue](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched-mlfq.pdf)
    * Chapter 9.7 The Linux Completely Fair Scheduler (CFS) in [Chapter 9 Scheduling: Proportional
      Share](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched-lottery.pdf)

## CPU Scheduling

* Sharing a core by multiple processes
* (Distributing processes across different cores is also a scheduling problem but that's not our
  focus in this course.)
* Context switch: stopping one process and starting/resuming another process.
    * This has overhead since it requires many complex steps to accomplish this. I.e., switching
      context frequently is not a good idea.
* Ready queue and I/O queue

  ```bash
  ------> ready queue --------------> CPU -> terminate
      |                                |
      \--- I/O done <--- I/O queue <---/
  ```

    * When a user launches `Process A`, it gets added to the ready queue.
    * At some point, the scheduler (a component inside the kernel) picks `Process A` and runs it.
    * If `Process A` calls a system call that requires I/O (e.g., a file read call), the scheduler
      puts `Process A` in the I/O queue and runs another process. (This is an example of context
      switch.) This way, we can keep the CPU busy with work.
    * When the I/O is done, the scheduler moves `Process A` from the I/O queue to the ready queue so
      it can run again.
    * At some point, the scheduler picks `Process A` from the ready queue and runs it.
* Scheduling == Picking one from the ready queue
* Types
    * Preemptive scheduling algorithms
        * A class of scheduling algorithms with the ability to stop a process any time and run
          another process.
    * Non-preemptive scheduling algorithms
        * A class of scheduling algorithms that do not stop processes at will. A running process
          gives up the core when it terminates or when it "waits" for something that takes a long
          time, e.g., I/O, child wait, sleep.

## Scheduling Criteria

* Maximize:
    * CPU utilization: keep the CPU as busy as possible
    * Throughput: # of processes that complete their execution per time unit
* Minimize
    * Turnaround time: amount of time to execute a particular process (time from submission to
      termination)
    * Waiting time: amount of time a process has been waiting in ready queue
    * Response time: amount of time it takes from when a request is submitted until the first
      response is produced

## Scheduling Algorithms

### First Come, First Served (FCFS)

* Run in the order of arrival.
* Non-preemptive
* Example
    * P1 takes 24 time units. P2: 3. P3: 3.
    * Suppose 3 processes arrive around the same time but gets added to the ready queue in the order
      of P1, P2, P3.
    * What is the waiting time?
        * P1 = 0; P2  = 24; P3 = 27
    * What is the average waiting time?
        * (0 + 24 + 27) / 3 = 17
    * Calculating the waiting time and the average waiting time evaluates your understanding.
    * Drawing a diagram like the following is a good idea to understand various scheduling
      algorithms.

      ```bash
      +------------------+
      | P1     | P2 | P3 |
      +------------------+
      0        24   27   30
      ```

* Problem?
    * A long process can sabotage all other processes.

### Shortest Job First (SJF)

* Let's try something where a long process doesn't sabotage all other processes.
* Among the remaining processes, pick the process with the shortest execution time.
* Assume for the sake of discussion, we know how long each process takes.
* Non-preemptive
* Example
    * Execution times
        * P1: 7, P2: 4, P3: 1, P4: 4
    * Arrival times
        * P1: 0, P2: 2, P3: 4, P4: 5
    * Steps
        * (Draw a timeline diagram yourself to better understand the steps.)
        * P1 runs until time 7.
        * P3 is the shorted job, so P3 runs next until it's done (time 8).
        * P2 and P4 are the next shorted jobs (tie). The scheduler can pick any one of them. Let's
          assume the scheduler picks P2. So P2 runs until it's done (time 12).
        * P4 is the next shorted job and runs until it's done (time 16).
    * Average waiting time?
        * (0 + 6 + 3 + 7) / 4  = 4

### Shortest Remaining Time First (SRTF)

* Assume for the sake of discussion, we know how long each process takes.
* Schedule the process with the shortest remaining execution time.
* This one's *preemptive*.
* Example
    * Let's use the same execution and arrival times as the above.
        * Execution times
            * P1: 7, P2: 4, P3: 1, P4: 4
        * Arrival times
            * P1: 0, P2: 2, P3: 4, P4: 5
    * Steps
        * (Draw a timeline diagram yourself to better understand the steps.)
        * At time 0, P1 arrives and runs.
        * At time 2, P2 arrives. Now, the scheduler needs to pick the one with the shortest
          remaining time. P1 has 5 and P2 has 4, so run P2.
        * At time 4, P3 arrives. Now, the scheduler needs to pick the one with the shorted remaining
          time. P1 has 5, P2 has 2, and P3 has 1, so run P3.
        * At time 5, P3 is done and P4 arrives. The scheduler picks P2 as it's the shortest
          remaining time one.
        * At time 7, P2 is done. The scheduler picks P4.
        * At time 11, P4 is done. The scheduler runs P1.
* This and SJF need to know how long processes take.
    * This is obviously a problem since we don't know the future.
* Side note: Interactive vs. batch
    * Interactive
        * Mainly user driven
        * Regular desktop applications
    * Batch
        * You run a program from start to end. No interaction in the middle.
        * Compiling a program
        * Data analytics
    * SJF & SRTF are mainly for batch processes (jobs). There are techniques developed to estimate
      how long a batch job takes, and we can use the estimates for scheduling.

### Round Robin

* Forget about knowing how long things take, let's just use round robin.
* Preemptive
* At every *x* time units (*quantum*), the scheduler picks the next process in the ready queue and
  runs it. "Next" is defined in the arrival order (just like FCFS). The stopped process becomes the
  last process in the ready queue (if it's not done).
* If the quantum is long, it's effectively the same as FCFS. If it is too short, the context switch
  overhead is too much, hence it is not going to have much time to do actual work, i.e., running
  processes.

### Priority Scheduling

* Among the processes in the ready queue, pick the process with the highest priority and run it.
* This can be either preemptive or non-preemptive.
* Motivation: real-time tasks with deadlines
    * There are certain systems that require hard or soft deadlines for their computational tasks.
      For example, an airplane controller needs to respond to an outside event (e.g., an incoming
      bird) within a short time period.
    * Hard deadlines are strict deadlines that *must not* be missed because the consequence of
      missing a deadline can be catastrophic. Soft deadlines are approximate deadlines that can be
      missed but should not be by much. Accordingly, there are hard real-time systems and soft
      real-time systems.
    * These tasks with deadlines are called *real-time* tasks. They should have higher priorities
      than other computational tasks, i.e., they are more important to run.
* A priority is typically expressed as a number (where a *smaller* number has a *higher* priority).
* Problem: starvation
    * Lower priority processes may never run. (Imagine a scenario where high priority processes keep
      arriving and a low priority process never gets a chance to run.)

### Multilevel Queue Scheduling

* This is a priority-based scheduling algorithm but avoids starvation.
* We group processes based on categories, e.g., foreground process category, background process
  category, system (OS) process category.
* Each category gets a priority value (separate from process priority values). E.g., the system
  processe category can have the highest priority (priority 0), the foreground category next
  (priority 1), and the background category the lowest (priority 2).
* The ready queue is partitioned into multiple queues, one queue per category. E.g., we could have
  one queue for system processes, another for foreground processes, and the third for background
  processes.

  ```bash
  +--------------------------+
  | system process queue     |
  +--------------------------+

  +--------------------------+
  | foreground process queue |
  +--------------------------+

  +--------------------------+
  | background process queue |
  +--------------------------+
  ```

* Each *queue* gets CPU time based on its priority. There are many algorithms one can use but for
  example, we could use the following algorithm (each number indicates which priority queue we
  choose):
    * 0, 1, 2, 0, 1, 0, 0, 1, 2, 0, 1, 0, 0, 1, 2, 0, 1, 0, ...
    * This is preemptive.
    * We first pick a process from priority 0 queue.
    * We preempt the process, and run a process from priority 1 queue.
    * We preempt and run a process from priority 2 queue.
    * We repeat this and go back to pick a process from priority 0 and then priority 1 queues.
      However, we stop there and don't run a process from priority 2 queue.
    * So we go back to pick a process from priority 0 queue. However, we stop there and don't run a
      process from priority 1 queue.
    * If we repeat the above steps, we give more CPU time for priority 0 queue than priority 1
      queue, and also more CPU time for priority 1 queue than priority 2 queue.
    * This is called *weighted* RR. It is a variation of RR but there is a weight associated with
      scheduling, i.e., it is basically RR but we give more CPU time to higher priority queues.
* When we pick a process from each queue, we also use a scheduling algorithm. We do not need to use
  the same scheduling algorithm for every queue. E.g., priority 0 queue runs the priority scheduling
  algorithm discussed above, priority 1 queue runs RR, and priority 2 queue runs the FCFS algorithm.
* This avoids starvation since every process eventually gets a chance to run.

### Multilevel Feedback Queue Scheduling

* Use multiple queues but move a process to lower priority queues if it takes too much CPU time.
* Example

  ```bash
        Q0
        +-------------+
    --> | quantum: 8  | --+
        +-------------+   |
                          |
    +---------------------+
    |
    |   Q1
    |   +-------------+
    +-> | quantum: 16 | --+
        +-------------+   |
                          |
    +---------------------+
    |
    |   Q2
    |   +-------------+
    +-> | FCFS        | -->
        +-------------+
  ```

    * Q0 and Q1 use RR. Q2 uses FCFS.
    * `Process A` enters Q0, and gets 8 ms (RR).
    * If it does not finish, move it to Q1.
    * Run all other processes in Q0.
    * When there is no more processes left in Q0, run processes in Q1 (e.g., `Process A`).
    * `Process A` gets 16 ms (RR).
    * If `Process A` is still not done, move it to Q2.
    * Run all other processes in Q0 and Q1.
    * When there is no more processes left in Q0 and Q1, run processes in Q2 (e.g., `Process A`).
    * Run `Process A` until it's done (FCFS).
    * This is one example, and one can use other scheduling algorithms for the queues. The key is
      lowering priority by moving to a lower priority queue if a process runs for too long. This is
      called process *aging*.

### Linux Completely Fair Scheduler (CFS)

* Linux categorizes processes into two classes.
    * Real-time processes (priority values 0 to 99)
    * Normal processes (priority values 100 to 139)
* We can use nice value to change/assign a priority for a normal process.
    * Nice values range from -20 to +19 (lower nice == higher priority).
    * The default nice value is 0.
    * The nice value of -20 maps to priority 100.
    * The nice value of +19 maps to priority 139.
* Linux CFS
    * Sorts all processes based on their run time (execution time). The longer it ran, the less
      chance it gets to run. This means that CFS tries to make sure that each process uses a similar
      amount of CPU time. The catch here is that it does this not based on physical (actual) run
      time but based on virtual run time (calculated with a formula).
    * Virtual run time = physical run time + (decay formula)
        * Higher decay with lower priority (i.e., the decay formula produces a bigger value for a
          lower priority).
        * I.e., given the same physical run time, a lower priority process appears to have used more
          CPU time.
