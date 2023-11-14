# Synchronization

* We generally use the term *synchronization* to refer to coordinating the execution among different
  threads. Often times, when using threads, careful coordination is necessary to avoid
  difficult-to-debug problems such as data races.
* There are a few synchronization primitives that we can use for coordinating the execution among
  threads, such as locks, condition variables, and semaphores.
* Reference chapters from OSTEP:
    * [Chapter 28 Locks](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf)
    * [Chapter 30 Condition Variables](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-cv.pdf)
    * [Chapter 31 Semaphores](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-sema.pdf)
    * [Chapter 32 Concurrency Bugs](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-bugs.pdf)
    * Keep in mind that OSTEP's discussions are much more in depth, which we consider out of scope
      for this course. Thus, you can use it as a reference but the main source should still be this
      lecture note.

## Locks (Mutexes)

* Consider the data race example discussed previously. Updating `cnt` is not a single operation but
  consists of multiple sub-operations, e.g., reading from memory, adding `1` to `cnt`, and writing
  it back to memory. Thus, if you have multiple threads updating `cnt` concurrently, these
  sub-operations run concurrently as well, which means that the sub-operations from different
  threads interleave and mix up with each other during their execution. This mix-up of
  sub-operations is the main source of unexpected/inaccurate results as discussed previously.
* To avoid this, we need to be able to prevent this mix-up of sub-operations from different threads.
* A lock or a mutex (for *mut*ual *ex*clusion) is a primitive that gives us this ability.
* Generally, a lock mechanism consists of the following three things.
    * A lock variable that defines a lock.
    * A `lock()` function that grabs a lock.
    * An `unlock()` function that releases a lock.
* For example, the pthread library provides a lock mechanism as follows.
    * `pthread_mutex_t`: a data type to define a lock.
    * `int pthread_mutex_lock(pthread_mutex_t *mutex)`: a lock function that grabs the lock passed
      as the argument.
    * `int pthread_mutex_unlock(pthread_mutex_t *mutex)`: an unlock function that releases the lock
      passed as the argument.
    * Other languages (e.g., Java, Python, etc.) provide similar lock mechanisms with different type
      names and function names.
* The central guarantee of a lock is that *only a single thread can hold a lock*.
    * Let's use pthread's lock mechanism as a concrete example.
    * Let's assume we have a lock named `mutex`, i.e., we have declared `pthread_mutex_t mutex`
      somewhere in our code.
    * If thread `A` calls `lock()` on `mutex`, i.e., `pthread_mutex_lock(&mutex)`, the underlying
      lock mechanism allows thread `A` to grab the lock `mutex`.
    * If thread `B` next calls `pthread_mutex_lock(&mutex)`, the underlying lock mechanism does not
      allow thread `B` to grab the lock `mutex` since thread `A` holds the lock already. The lock
      mechanism allows only a single thread to grab a lock. This is the central guarantee provided
      by the lock mechanism. Now, the lock mechanism *blocks* thread `B` from executing further and
      makes it *wait* until the lock `mutex` becomes available to grab.
    * What blocking means here is that, when thread `B` calls `pthread_mutex_lock(&mutex)`, the call
      *does not return* until `mutex` becomes available to grab.
    * In summary, calling `pthread_mutex_lock(&mutex)` either grabs the lock `mutex` (if no other
      thread holds the lock) or gets blocked and waits (if there is a thread that holds the lock).
    * If thread `A` (that holds the lock `mutex`) calls `unlock()`, i.e.,
      `pthread_mutex_unlock(&mutex)`, the underlying lock mechanism releases the lock and makes it
      available for another thread to grab.
* Even if there are more than two threads, only a single thread can hold a lock and all other
  threads need to wait (if they all call `lock()`) until the lock becomes available.
    * Only one thread at a time can grab a lock and we *can't* control the order in which different
      threads grab the lock. It depends on the underlying lock mechanism.
* Going back to the data race example, we can use a lock to avoid the mix-up of sub-operations from
  different threads.
    * We can "guard" the region of code that updates `cnt` by calling `lock()` beforehand and
      calling `unlock()` afterward. This way, only a single thread can execute the code and safely
      update `cnt` without worrying about sub-operations getting mixed-up from different threads.
* The following code demonstrates the data race problem.

  ```c
  #include <pthread.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>

  int cnt = 0;

  static void *thread_func(void *arg) {
    for (int i = 0; i < 10000000; i++)
      cnt++;
    pthread_exit(0);
  }

  int main(int argc, char *argv[]) {
    pthread_t t1;
    pthread_t t2;

    if (pthread_create(&t1, NULL, thread_func, NULL) != 0)
      perror("pthread_create");

    if (pthread_create(&t2, NULL, thread_func, NULL) != 0)
      perror("pthread_create");

    if (pthread_join(t1, NULL) != 0)
      perror("pthread_join");
    if (pthread_join(t2, NULL) != 0)
      perror("pthread_join");

    printf("%d\n", cnt);

    exit(EXIT_SUCCESS);
  }
  ```

* The following code uses a lock to properly guard the updating of `cnt`. Notice how the lock
  `mutex` is initialized with the initialization macro, `PTHREAD_MUTEX_INITIALIZER`.

  ```c
  #include <pthread.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>

  int cnt = 0;
  pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

  static void *thread_func(void *arg) {
    for (int i = 0; i < 10000000; i++) {
      pthread_mutex_lock(&mutex);
      cnt++;
      pthread_mutex_unlock(&mutex);
    }
    pthread_exit(0);
  }

  int main(int argc, char *argv[]) {
    pthread_t t1;
    pthread_t t2;

    if (pthread_create(&t1, NULL, thread_func, NULL) != 0)
      perror("pthread_create");

    if (pthread_create(&t2, NULL, thread_func, NULL) != 0)
      perror("pthread_create");

    if (pthread_join(t1, NULL) != 0)
      perror("pthread_join");
    if (pthread_join(t2, NULL) != 0)
      perror("pthread_join");

    printf("%d\n", cnt);

    exit(EXIT_SUCCESS);
  }
  ```

## Lock Usage

* Atomicity
    * Using a lock on a critical section makes all operations within the critical section *atomic*,
      i.e., all operations would run as if they were a single operation. Going back to the previous
      data race example, if we used a lock for updating `cnt`, all sub-operations (reading from
      memory, updating `cnt`, and writing back to memory) would run without getting any interference
      from other threads' sub-operations. In other words, they would run as if they were a single
      step.
    * Atomicity is sometimes referred to as *all or nothing* as it runs either all operations or no
      operations at all.
* Serialization and interleaving
    * Using a lock effectively *serializes* operations, i.e., only one thread at a time can run the
      operations guarded by a lock.
    * Operations from different threads are *interleaved* in some order. But we can't control the
      order in which different threads run.
* Protecting shared variables
    * Whenever multiple threads share a variable, we need to use a lock to control the access to the
      variable in order to avoid data race problems. Typically in C, a shared variable is a global
      variable.
    * We can also use other synchronization primitives that we will discuss later.
* Multiple locks
    * We do not need to use only one lock to protect shared variables.
    * For example, suppose we have four threads where two threads share a global variable `a` and
      the other two threads share another global variable `b`. We can use two locks, one for `a` and
      another for `b`, to control access. If we use only a single lock, four threads will compete
      for the same lock even if they do not all access the same variable.
    * Reducing `lock contention` is important for performance.
* `pthread_mutex_trylock()` and `pthread_mutex_timedlock()`
    * These functions allows us to control the blocking behavior of `pthread_mutex_lock()`.
    * `pthread_mutex_trylock()`: from the man page, "The `pthread_mutex_trylock()` function shall be
      equivalent to `pthread_mutex_lock()`, except that if the mutex object referenced by mutex is
      currently locked (by any thread, including the current thread), the call shall return
      immediately."
    * `pthread_mutex_timedlock()`: from the man page, "The `pthread_mutex_timedlock()` function
      shall lock the mutex object referenced by mutex. If the mutex is already locked, the calling
      thread shall block until the mutex becomes available as in the `pthread_mutex_lock()`
      function. If the mutex cannot be locked without waiting for another thread to unlock the
      mutex, this wait shall be terminated when the specified timeout expires."

## Critical Section (CS) and Thread Safety

* Critical Section (CS)
    * A critical section is a term that is commonly used in concurrent programming.
    * From OSTEP, "A critical section is a piece of code that accesses a shared variable (or more
      generally, a shared resource) and must not be concurrently executed by more than one thread."
    * If a thread is executing the CS, no other threads should execute the CS.
* An ideal solution for the CS problem must satisfy three requirements:
    * *Mutual exclusion*: Only one thread should be allowed to run in the CS
    * *Progress*: A thread should eventually complete (i.e., make progress).
    * *Bounded waiting*: An upper bound must exist for the amount of time a thread waits to enter
      the CS (i.e., a thread should only be blocked for a finite amount of time).
* Thread safety
    * A *thread safe* function is a function that multiple threads can run safely.
    * A thread safe function either does not access shared resources or provides proper protection
      for its critical sections that access shared resources.
    * Reentrant vs nonreentrant functions
        * Reentrant functions don't use any shared (global or static) variables, thus thread safe.
        * A common technique to implement a reentrant function: any info returned to the caller or
          maintained across function calls should use caller-allocated buffers. E.g., when calling
          `write()`, we need to allocate a buffer and pass it to the function. This is a common
          technique to make a function reentrant.

## Deadlock and Livelock

* Deadlock
    * A deadlock is a condition where a set of threads each hold a resource and wait to acquire a
      resource held by another thread.
    * The threads get stuck and make no progress.
* Activity
    * Write a program that creates two threads and two locks.
    * Each thread should request two locks in sequence but in a different order.
    * The following code demonstrates this and it could deadlock. (Deadlocks don't always occur due
      to the timing of thread execution. Thus, the following code may not always deadlock. But
      still, it does occur from time to time, which is a problem.)

      ```c
      #include <pthread.h>
      #include <stdio.h>
      #include <stdlib.h>
      #include <unistd.h>

      static pthread_mutex_t mutex0 = PTHREAD_MUTEX_INITIALIZER;
      static pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;

      static void *thread0(void *arg) {
        pthread_mutex_lock(&mutex0);
        printf("thread0: mutex0\n");
        pthread_mutex_lock(&mutex1);
        printf("thread0: mutex1\n");
        pthread_mutex_unlock(&mutex0);
        pthread_mutex_unlock(&mutex1);
        pthread_exit(0);
      }

      static void *thread1(void *arg) {
        pthread_mutex_lock(&mutex1);
        printf("thread1: mutex1\n");
        pthread_mutex_lock(&mutex0);
        printf("thread1: mutex0\n");
        pthread_mutex_unlock(&mutex1);
        pthread_mutex_unlock(&mutex0);
        pthread_exit(0);
      }

      int main() {
        pthread_t t0;
        pthread_t t1;

        if (pthread_create(&t0, NULL, thread0, NULL) != 0)
          perror("pthread_create");

        if (pthread_create(&t1, NULL, thread1, NULL) != 0)
          perror("pthread_create");

        if (pthread_join(t0, NULL) != 0)
          perror("pthread_join");
        if (pthread_join(t1, NULL) != 0)
          perror("pthread_join");

        exit(EXIT_SUCCESS);
      }
      ```

* A deadlock may arise when the following conditions hold at the same time.
    * *Hold and wait*: threads are already holding resources but also are waiting for additional
      resources being held by other threads.
    * *Circular wait*: there exists a set {T0, T1, ..., Tn} of threads such that T0 is waiting for
      a resource that is held by T1, T1 is waiting for T2, ..., Tn–1 is waiting for T0.
    * *Mutual exclusion*: threads hold resources exclusively.
    * *No preemption*: resource released only voluntarily by the thread holding it
* The above code satisfies these conditions since the `t0` could hold `mutex0` and request `mutex1`,
  while `t1` could hold `mutex1` and request `mutex0`.
* Deadlock prevention: ensuring at least one of the conditions cannot hold.
    * If we can break one of the four conditions from above, we can prevent a deadlock from
      occurring.
* Two potential techniques.
    * Grabbing all locks at once atomically: this breaks the hold-and-wait condition since you grab
      all the locks together or no locks at all. The following code uses another lock to achieve
      this, using the same example from above.

      ```c
      #include <pthread.h>
      #include <stdio.h>
      #include <stdlib.h>
      #include <unistd.h>

      static pthread_mutex_t mutex0 = PTHREAD_MUTEX_INITIALIZER;
      static pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;
      static pthread_mutex_t another_lock = PTHREAD_MUTEX_INITIALIZER;

      static void *thread0(void *arg) {
        pthread_mutex_lock(&another_lock);
        pthread_mutex_lock(&mutex0);
        printf("thread0: mutex0\n");
        pthread_mutex_lock(&mutex1);
        printf("thread0: mutex1\n");
        pthread_mutex_unlock(&mutex0);
        pthread_mutex_unlock(&mutex1);
        pthread_mutex_unlock(&another_lock);
        pthread_exit(0);
      }

      static void *thread1(void *arg) {
        pthread_mutex_lock(&another_lock);
        pthread_mutex_lock(&mutex1);
        printf("thread1: mutex1\n");
        pthread_mutex_lock(&mutex0);
        printf("thread1: mutex0\n");
        pthread_mutex_unlock(&mutex1);
        pthread_mutex_unlock(&mutex0);
        pthread_mutex_unlock(&another_lock);
        pthread_exit(0);
      }

      int main() {
        pthread_t t0;
        pthread_t t1;

        if (pthread_create(&t0, NULL, thread0, NULL) != 0)
          perror("pthread_create");

        if (pthread_create(&t1, NULL, thread1, NULL) != 0)
          perror("pthread_create");

        if (pthread_join(t0, NULL) != 0)
          perror("pthread_join");
        if (pthread_join(t1, NULL) != 0)
          perror("pthread_join");

        exit(EXIT_SUCCESS);
      }
      ```

    * Acquiring locks in the same global order for all threads: this also breaks the circular wait
      condition as all threads try to grab locks in the exact same order.

      ```c
      #include <pthread.h>
      #include <stdio.h>
      #include <stdlib.h>
      #include <unistd.h>

      static pthread_mutex_t mutex0 = PTHREAD_MUTEX_INITIALIZER;
      static pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;

      static void *thread0(void *arg) {
        pthread_mutex_lock(&mutex0);
        printf("thread0: mutex0\n");
        pthread_mutex_lock(&mutex1);
        printf("thread0: mutex1\n");
        pthread_mutex_unlock(&mutex0);
        pthread_mutex_unlock(&mutex1);
        pthread_exit(0);
      }

      static void *thread1(void *arg) {
        pthread_mutex_lock(&mutex0);
        printf("thread1: mutex0\n");
        pthread_mutex_lock(&mutex1);
        printf("thread1: mutex1\n");
        pthread_mutex_unlock(&mutex0);
        pthread_mutex_unlock(&mutex1);
        pthread_exit(0);
      }

      int main() {
        pthread_t t0;
        pthread_t t1;

        if (pthread_create(&t0, NULL, thread0, NULL) != 0)
          perror("pthread_create");

        if (pthread_create(&t1, NULL, thread1, NULL) != 0)
          perror("pthread_create");

        if (pthread_join(t0, NULL) != 0)
          perror("pthread_join");
        if (pthread_join(t1, NULL) != 0)
          perror("pthread_join");

        exit(EXIT_SUCCESS);
      }
      ```

* Livelock
    * A livelock is a condition where a set of threads each execute instructions actively, but they
      still don't make any progress.
    * For example, suppose we have two threads, T0 and T1, each of which attempts to acquire two
      resources, R0 and R1. They run a function that acquires the first resource (either R0 or R1),
      prints a message, then tries to acquire the second resource. If the second resource is not
      available, the function releases the first resource and tries again. Given this function,
      consider a scenario where T0 and T1 runs concurrently. T0 acquires R0 and T1 acquires R1. T0
      then tries to acquire R1 and T1 tries to acquire R0. However, since R0 and R1 are not
      available, T0 releases R0 and tries again, and T1 releases R1 and tries again. T0 and T1
      repeats this sequence forever.
    * In this example, both T0 and T1 actively execute the function but they do not make any
      progress. This is called a livelock.
    * Both deadlocks and livelocks do not make any progress. In a livelock scenario, threads do
      still execute. In a deadlock scenario, threads are stuck and do not execute anything actively.
