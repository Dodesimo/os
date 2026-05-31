- non-deadlock bugs: atomicity violation and order violation
	- atomicity violation
		-  ```c
	  if (thd->proc_info) {
		  fputs(thd->proc_info, ...)
	  }//thread 1
	  thd->proc_info = NULL //thread 2
		  ```
		* issue: if first thread gets interrupted and second thread starts running, second thread could set field to null and cause a crash
		* solution: have a shared lock that surrounds both areas
		* ```c
		  pthread_mutex_lock(&proc_inf_lock);
		  if (thd->proc_info) {
			  fputs(thd->proc_infom, ...)
		  }
		  pthread_mutex_unlock(&proc_inf_lock);
		  //thread 1
		  
		  pthread_mutex_lock(&proc_inf_lock)
		  thd->proc_info = NULL
		  pthread_mutex_unlock(&proc_inf_lock);
		  ```
	* order-violation bugs:
		* ```c
		  void init() {
			  mThread = PR_CreateThread(mMain, ...);
		  }
		  //thread one
		  
		  void mMain(...) {
			  mState = mThread->State;
		  }
		  ```
		* second thread assumes that mThread is initialized
		* best way to solve this:
			* have a mutex and conditional variable
			* ```c
			  void init() {
				  mThread = PR_CreateThread(mMain, ...);
				  pthread_mutex_lock(&mtLock);
				  mtInit = 1;
				  pthread_cond_signal(&mtCond);
				  pthread_mutex_unlock(&mtLock);
			  }
			  //thread one
		  
			  void mMain(...) {
				  pthread_mutex_lock(&mtLock);
				  while (mtInit != 1) {
					  pthread_cond_wait(&mtCond, &mtLock);
				  }
				  mState = mThread->State;
				  pthread_mutex_unlock(&mtLock);
			  }
			  ```
* deadlock:
	* thread 1: holding Lock 1 and waiting for Lock 2
	* thread 2: holding Lock 2 and waiting for Lock 1
	* deadlocks happen due to complex dependencies between components
		* also happens due to encapsulation that makes modularity a priority (divide complex systems into various subcomponents)
* conditions for deadlock:
	* mutual exclusion: threads claim exclusive control over resources
	* hold and wait: threads hold resources allocated to them while waiting for additional resources
	* no preemption: resources can't be forcefully removed from threads
	* circular wait: circular chain of threads, each thread holds one or more resources being requested by the next thread in the chain
* prevention:
	* break circular waits
		* total ordering on lock acquisition: two locks, always acquire Lock 1 over Lock 2
		* if complex code: have a partial ordering
		* can also enforce lock ordering through lock address:
			* given function `do_something(mutex_t *m1, mutex_t *m2)`
				* `if (m1 > m2) {pthread_mutex_lock(m1); thread_mutex_lock(m2);}`
				* `else {pthread_mutex_lock(m2); thread_mutex_lock(m1);}`
				* always enforce highest address order to lowest order locking
	* hold and wait requirement:
		* acquire all locks in an atomic fashion:
			* ```c 
			  pthread_mutex_lock(prevention); //gather all locks at the same time
 			  pthread_mutex_lock(L1); // can enforce order here
			  pthread_mutex_lock(L2);
			  pthread_mutex_lock(L3);
			  pthread_mutex_unlock(prevention); 
			  ```
			* code guarantees no untimely switch can happen in lock acquisition
			* so no thread holds onto partial resources while waiting for another
		* issues w/ this approach:
			* anti encapsulation: calling a routine, approach requires us to know what locks must be held at time
			* decrease concurrency: all possible locks get acquired at the same time
	* no preemption:
		* multiple lock acquisitions cause problems because waiting for one, we are holding another
		* flexible interface: `pthread_mutex_trylock()`: 
			* grabs lock if available or returns an error code indicating a held lock
			* ```c
			  pthread_mutex_lock(L1);
			  if (pthread_mutex_trylock(L2) != 0) {
				  pthread_mutex_unlock(L1); //on a failiure, let go of other resoruce we are holding
			  }
			  ```
			*  results in livelock: two threads can both be repeatedly attempting a sequence and failing to acquire both locks
				* keeps happening, wasting CPU cycles
			* prevent this by having exponential backoff or some delay
			* have to be very clean w/ the way we let go (other allocated memory before the try lock failure needs)
			* doesn't really add preemption of forcefully taking lock away from thread, but rather allows to back out in a graceful way
	* mutual exclusion:
		* built lock-free data structures
		* use powerful hardware instructions to build data structures w/o explicit locking
		* use an atomic hardware instruction like `CompareAndSwap`
			* ```c
			  void AtomicIncrement(int *value, int amount) {
				  do {
					  int old = *value;
				  } while (CompareAndSwap(value, old, old + amount) == 0) //keep getting the current value if we the value isn't equal to the old, if its equal, then set it to old + amount
			  }
			  ```
		* can also be applied to linked lists
			* ```c
			  do {
				  n->next = head;
			  } while (CompareAndSwap(&head, n->next, n) == 0); //send the next of this current node to the head and then finally compare and swap will set the head to n (keep running this as we fail)
			  ```
* avoidance:
	* have global knowledge on which locks threads grab and schedule threads such that we avoid dead lock
		* if two threads require the same two locks, the scheduler can choose to just not run them at the same time
		* grabbing only one lock is fine (other thread just goes to sleep)
		* this may result in threads that require the same locks from running on the same processor
		* fear of deadlock results in less performance and limited concurrency, so avoidance isn't a widely used solution
	* detect and recover:
		* deadlock detection through a resource graph and checking for cycles
		* event of a cycle, system restarts