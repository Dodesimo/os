- thread: new abstraction, multi-threaded program has more than one point of execution 
	- each thread is like a separate process but they share the same address space and access the same data
- each thread has:
	- program counter
	- private set of registers (switching between threads requires a context switch)
- context switch between threads similar to processes: register state gets saved, register state of other thread gets restored
	- all threads have state saved to Thread Control Block (TCB)
	- but address space remains same (page table isn't switched)
- a process w/ one thread: has a single stack at the bottom
- multi-threaded process, each thread runs independently
	- each thread gets own stack
	- thread-local storage: stack-allocated variables, parameters, return values
- why threads?
	- parallelism
		- large array program, if running on a single process, just do the operation
		- multiple processors, speed up work by using each processor to do a portion of the work
		- use a thread per processor to do this work and run program faster through parallelization
	- blocking program progress due to slow I/O
		- program does different types of I/O and needs to wait for responses
		- instead of waiting, program does something else
		- threads allow us to do this: while one thread waits (blocked, waiting for I/O), CPU scheduler switches to other threads and does something useful
		- enables overlap of I/O within single program, while multiprogramming enables overlap across programs for processes
	- above could be done w/ processes instead of threads
		- but threads share address space, and easy to share data 
		- processes make more sense for separate tasks with little sharing of data structures required
- `pthread_join()`: waits for a particular thread to complete (called twice), so both threads are waited for completion, and then main thread runs
	- ```c
	  pthread_t p1;
	  pthread_create(&p1, NULL, mythread, "A"); // mythread is function name
	  ```
* can't assume that the thread created first will run first (this is because of the way the scheduler behaves)
	* threads can run in any arbitrary order after their creation
	* code within the threads will be ran in a continuous/synchronous fashion
* suppose we had a global shared variable, and we want threads to increment it
	* issue: not atomic
	* series of assembly instructions
	* as one thread attempts to run it, it gets kicked off due to timer interrupt and new thread gets scheduled
		* this thread is able to execute instructions and counter gets incremented
	* when first thread comes back, it will leave off where it was (doesn't increment and just resaves the counter value)
		* so counter doesn't get incremented
		* need a way for thread to do all of these instructions at once before being rescheduled
* this is a data race/race condition: multiple threads manipulating a shared variable without any sort of synchronization
	* causes indeterminate results
* critical section: piece of code accessing a shared variable, can't be concurrently executed by more than one thread
* want mutual exclusion: one thread executing in the critical section, others are prevented
* want an atomic operation
	* do everything needed to be done in a single step
	* either it happens or it doesn't
	* do so through synchronization primitives
		* enable reliable results from code
		* requires hardware support and OS support
* another interaction b/w threads: one thread waits for another to complete before it does something
	* process performs disk i/o, process goes to sleep, process needs to be woken
* critical section: code that accesses shared resource
* race condition:  multiple threads enter a critical section at the same time
* indeterminate program: one or more race conditions, output of program varies from run to run, depending on when threads are ran
	* outcome isn't deterministic
* to avoid this: threads must use mutual exclusion primitives, guarantees only a single thread enters critical section and avoids races