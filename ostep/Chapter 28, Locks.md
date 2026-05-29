- execute series of instructions atomically, but interrupts on a single processor we can't
	- use a lock to ensure that a critical section is a single atomic instruction
- lock holds state of lock at any instant
	- available/unlocked/free: no thread holds it
	- acquired/locked/held: one thread holds the lock and is in the critical section
	- other info like what thread holds lock, queue for ordering lock acquisition
- basic lock:
	- if another thread tries to hold the lock, it spins while waiting for access
	- when lock is freed, waiting threads are notified of change, first one gets the lock, and enters the critical section
- gives a guarantee that only one thread can be active within that code
- posix defines lock as a mutex: mutual exclusion between threads, only one thread is in the critical section, excludes others from entering till done
- `pthread_mutex_lock` takes variables to lock and unlock (different locks protect different variables)
	- increases concurrency through fine-grained locking approach
- how to evaluate a lock:
	- does each thread contending for lock get a fair shot at acquisition
		- in other words does a thread contending for the lock starve and not obtain it
	- does the lock actually enable mutual exclusion
	- is the lock performant?
		- does the overheads add up?
		- no contention: overhead of grabbing/releasing lock
		- multiple threads content for a single CPU: performance concerns?
- ways to lock:
	- most basic: disable interrupts for critical sections, enable them after leaving
	- thread can be sured that code executes will execute, no other thread interferes
	- issues:
		- too much trust (we ask processor to do a privileged operation and trust that facility isn't abused)
		- disabling interrupts only disables for that processor not others (so this wouldn't work on multicore systems because a thread running on another CPU doesn't care about disabled interrupts on another CPU)
		- interrupts could get lost for extended periods of times, results in systems problems (how dyk if disk device finished a request?)
	- simple load/store:
		- ```c
		  void lock (lock_t *mutex) {
			  while (mutex->flag == 1);
			  mutex->flag = 1;
		  }
		  void unlock (lock_t *mutex) {
			  mutex-flag = 0;
		  }
		  ```
		* issue: because this isn't atomic, run into a situation where both threads end up setting the flag to 1
		* poor performance: thread endlessly checks value of flag through spin-waiting
			* wastes time for another thread to release a lock, waste is super high on uniprocessor (other thread can't even run till a context switch)
	* use test and set instruction (atomic exchange):
		* algorithms like Peterson's algorithm don't need atomic hardware but don't work with relaxed memory consistency models
		* ```c++
		  void lock (lock_t *lock) {
			  while (TestAndSet(&lock->flag, 1) == 1); //while the old value that we get is 1, keep spinning, else we atomically set it to 1
		  }
		  void unlock(lock_t *lock) {
			  lock->flag = 0;
		  }
		  ```
		* we make the test part (returning the old lock value) and the set part (placing the new value) in one atomic operation (both must run), only one thread acquires lock
		* spin lock: uses CPU cycles until lock becomes available, to work, requires a preemptive scheduler (interrupts thread through timer to run a different thread)
	* spin lock effectiveness:
		* actually works
		* not fair, thread spinning could spin forever under contention (lead to starvation)
		* performance: 
			* threads competing for lock on single processor: performance overheads suck (thread holding lock preempted in a critical section, all other threads are chosen to run, are spinning in acquiring the lock, then waste CPU cycles)
			* multiple CPUs, ideally same # as threads:
				* Thread A and Thread B contend for a lock, but run on different CPUs
				* Thread A has the lock, B then spins
				* lock then available, B gets it, so not as many cycles are wasted
	* compare and swap:
		* ```c
		  int CompareAndSwap(int *ptr, int expected, int new) {
			  int original = *ptr;
			  if (original == expected) {
				  *ptr = new;
			  }
			  return original;
		  }
		  ```
		* test whether value at address is equal to expected, if so set it to new value
		* then return original
	* load-linked:
		* typical load instruction, fetches value from memory and places it in a register
			* ```c++
			  int LoadLinked(int *ptr) {
				  return *ptr;
			  }
			  ```
	* store-conditional:
		* only succeeds + updates value stored at address if no intervening store took place
		* ```c++
		  int StoreConditional(int *ptr, int value) {
			  if (no update to ptr since the load link) {
				  *ptr = value;
				  return 1; // set to the value, return 1 for success
			  } else {
				  return 0; //failed
			  }
		  }
		  ```
		* so only update the store if a load linked hasn't happened recently
		* ```c
		  void lock(lock_t *lock) {
			  while (true) {
				  while (LoadLinked(&lock->flag) == 1); //spin until we get a 0
				  if (StoreConditional(lock, 1) == 1) {
					  return; //this means that there weren't updates made, and we  set the value to 1
				  }
			  }
		  }
		  ```
		* why does this work?
			* if thread 1 does load-linked and gets a 0 and is about to store conditional but then is interrupted and thread 2 gets scheduled and it also gets a 0, and thread 1 is about to be rescheduled, it sets the lock to 1 and returns
			* when thread 2 gets rescheduled and is about to do `StoreConditional`, fails because it had just been set
	* fetch and add:
		* atomically increments value while returning old value at particular address
		* ```c
		  int fetchAndAdd(int *ptr) {
			  int oldValue = *ptr;
			  *ptr = *ptr + 1;
			  return oldValue;
		  }
		  ```
		* used to implement ticket lock:
			* thread wants lock: does fetch and add on ticket value (value is thread's turn)
			* when that is equal to thread's turn, thread can enter the critical section
				* unlock done by incrementing turn so next waiting thread can enter critical section
* spinning:
	* wastes clock cycles + entire time slice gets wasted doing nothing but checking value that's not going to check
	* solutions:
		* when you're going to spin, yield the CPU to another thread
		* ```c
		  void lock() {
			  while (TestAndSet(&flag, 1) == 1) {yield();}
		  }
		  ```
		* `yield()`: moves caller from running state to ready state, yielding thread ends up descheduling itself
		* this works on a uniprocessor (yields CPU and other thread can run and finish)
		* 100 other threads, 99 executes run and yield pattern, but this can cause a lot of latency in terms of context switching
			* we are assuming no queues, so random/some metric-defined threads get picked 
		* better approach: use a queue and sleep instead of spinning:
			* `park()`: puts a calling thread to sleep
			* `unpark(threadId)`: wakes up a particular thread designated by `threadId`
			* ```c
			  void lock(lock_t *m) {
				  while(TestAndSet(&m->guard, 1) == 1); //guard is a spinning lock for the state of the lock
				  if (m->flag == 0) {
					  m->flag = 1; //we get the lock
					  m->guard = 0;
				  } else {
					  queue_add(m->q, gettid());
					  m->guard = 0; //could cause a race condition here where B gets interrupted before park, A comes back, unlocks, unparks B, B then parks (continues to the next line), so no one can wake up B
					  park(); //use setpark before (thread indicates about to park, happens to be interrupt and another thread calls unpark, park returns immediately)
				  }
			  }
			  void unlock(lock_t *m) {
				  while (TestAndSet(&m->guard, 1) == 1);
				  if (queue_empty(m->q)) {
					  m->flag = 0;
				  } else {
					  unpark(queue_remove(m->q)); //flag remains 1, so the top of this queue gets access to the lock (can't access the guard anyway)
				  }
				  m->guard = 0;
			  }
			  ```
		* spin locks: can cause issues w/ scheduling:
			* low priority process gets lock, descheduled for high priority process
			* high priority process can't do anything because low priority process has the lock (so spins)
				* priority inversion because low priority process must be ran
			* solution:
				* temporarily boost lower thread's priority and let it run and overcome inversion (priority inheritance)
				* or ensure all threads have the same priority
	* linux has futex:
		* has more kernel functionality
		* each futex has a specific physical memory location, plus a per futex in-kernel queue
		* call to `futex_wait(address, expected)` puts calling thread to sleep if address == expected
		* call `futex_wake(address)`: wakes one thread in the waiting queue
		* integer: if negative means lock is held, high bit is set
	* two phase locks:
		* spins for a while
		* but if lock not acquired during first spin phase, second phase entered
			* caller put to sleep, woken up when lock becomes free later
