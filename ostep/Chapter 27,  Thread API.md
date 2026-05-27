- creating threads:
	- `pthread_create(thread, attr, start_routine, arg)`
	- `thread`: pointer to type `pthread_t`, initialize it w/ this
	- `attr`: specify attributes of this thread (like stack size or scheduling priority)
	- `start_routine`: function pointer, where should the function be running in (function name)
	- `arg`: void pointer, pass in any type of argument, return any type of result
- wait for completion: `pthread_join`
	- first argument: what thread to wait for (not a pointer the actual thread, so of type `pthread_t`)
	- second argument: pointer to where to store output of function call (of type `void **`)
	- don't return local variable from function (this will be deleted at the end of the thread's life so you're passing back a deallocated variable)
-  not all multi-threaded code uses join
	- may create a number of worker threads, use main thread to accept requests and pass them to workers (no join)
- parallel program that uses threads to do particular task in parallel: uses join to ensure all work done before exiting or doing next stage of computation
- locks:
	- `int pthread_mutex_lock(pthread_mutex_t *mutex)`
	- `int pthread_mutex_unlock(pthread_mutex_t *mutex)`
	- surround a critical section w/ this lock and then unlock
		- pass addresses to the mutex
	- if no thread holds a lock when the lock function called, thread gets lock, enters critical section
	- if another thread holds lock, thread getting lock doesn't return until its got the lock
	- many threads can be stuck waiting for a lock 
		- only thread w/ lock should call unlock
	- initialize a lock: use `PTHREAD_MUTEX_INITIALIZER` or `pthread_mutex_init()`
		- returns an integer for the second, check if its 0 for success
		- also do a `pthread_mutex_destroy()`
	- other  important programs:
		- `int pthread_mutex_trylock(pthread_mutex_t *mutex)` 
			- returns a failure if lock is held
		- `int pthread_mutex_timedlock(pthread_mutex_t *mutex, struct timespec *abs_timeout)`
			- returns after time out or after acquiring lock (w/ 0 degens to first)
	- condition variable: used for signaling between threads
		- `int pthread_cond_wait (pthread_cond_t *cond, pthread_mutex_t *mutex)`
			- first routine, puts calling thread to sleep and lets go of the lock
		- `int pthread_cond_signal(pthread_cont_t *cond)`
			- moves threads from conditional variables back into lock queue
		- lock should be held when calling these functions
		- ```c
		  pthread_mutex_lock(&lock);
		  while (ready == 0) {
			  pthread_cond_wait(&cond, &lock);
		  }
		  pthread_mutex_unlock(&lock);
		  
		  //other code
		  pthread_mutex_lock(&lock);
		  read = 1;
		  pthread_cond_signal(&cond);
		  pthread_mutex_unlock(&lock);
		  ```
  * keep code simple, init. lock and conditional variables, minimize thread interactions, check return codes, don't pass reference to variable on stack, each thread has its own stack (so stack stuff is private, to share stuff,  use heap or some other globally accessible place), use conditional variables for signaling b/w threads
  * 