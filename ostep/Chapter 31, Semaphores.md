- can be used as both locks and condition variables (so serves both roles)
- semaphores: 
	- obj. w/ integer value that can be manipulated w/ two routines
	- `sem_wait()` and `sem_post()`
- `sem_init(&s, 0, 1)`: 
	- second argument indicates semaphore shared between threads of the same process
	- third argument indicates init value
- what does either do:
	- ```c
	  int sem_wait(sem_t *s) {
		  //decrement semaphore value by one, wait if value of semaphore is negative (only sleep if < 0)
	  }
	  int sem_post(sem_t *s) {
		  //increment semaphore by one
		  //if there are one or more threads waiting, wake one up
	  }
	  ```
* binary semaphore:
	* simply used as a lock
	* ```c
	  sem_t m;
	  sem_init(*m, 0, 1); //one resource available to gain access to critical section
	  sem_wait(&m);
	  //gain access to the critical section (if the resource was 0, this owuldn't work because it would become negative inside the semaphore)
	  sem_post(&m);
	  ```
	* situation where thread 0 holds the lock, another thread tries to enter critical section by calling `sem_wait()`
		* thread 1 decrements value of semaphore to -1, sees its negative and waits (yields CPU)
		* when thread 0 finishes, calls `sem_post(&m)`, increments value of semaphore, restoring it back to 1
* semaphores: also useful to order events in a concurrent program (so like a condition variable)
	* thread waits for for something to happen
	* ```c
	  void *child (void *arg) {
		  sem_post(&s); //increase semaphore value by one, signal threads if waiting
		  return NULL;
	  }
	  
	  void main (int argc, char *argv[]) {
		  sem_init(&s, 0, X); //X should represent 0 so that the parent is initially sleeping, the child runs, the semaphore gets incremented to 0
		  pthread_t c;
		  pthread_create(&c, NULL, child, NULL);
		  sem(&wait); //as long as the underlying value is negative, wait		  
	  }
	  ```
	* if parent runs, sleeps after decrementing the semaphore to -1, child runs, increments it to 0, parent gets to run again
	* if child runs, increments the semaphore to 1, child runs, increments it to 0, parent runs again
* how to know what semaphore value to initialize
	* consider # of resources willing to give away immediately after initialization 
	* locks: 1 because ur giving away the lock
	* ordering: nothing to give away at the start (only when something gets terminated)
* producer/consumer problem (bounded buffer)
	* consumer: decrements full (initialized to 0), so sleeps
	* the full variable initialized to max resources, gets decremented producer puts data value into first entry of buffer, calls `sem_post(&full)` waking up a blocked consumer
	* potential race condition:
		* two producers calling put at same time, Pa fills first buffer entry, gets interrupted, producer Pb runs and also puts its data in the buffer
			* data gets overridden
			*  same thing can be applied to the consumers (in terms of taking multiple items at once)
		* ```c
		  void *producer (void * arg) {
			  int i;
			  for (i = 0; i < loops; i++) {
				  sem_wait(&empty);
				  sem_wait(&mutex); //reason why mutex is here b/c if it was before, if producer went to sleep if empty, consumer tries to get mutex but its not relinquished so deadlock. by having wait before, check condition and go to sleep if certain condition is met (don't worry about releasing mutex)
				  put(i);
				  sem_post(&mutex);
				  sem_post(&full);
			  }
		  }
		  
		  void *consumer (void *arg) {
			  int i;
			  for (i = 0; i < loops; i++) {
				  sem_wait(&full);
				  sem_wait(&mutex);
				  int tmp = get();
				  sem_post(&mutex); 
				  sem_post(&full);
			  }
		  }
		  ```
* read/writer locks:
	* allow lookups to be independent of writes (lookup doesn't alter state)
	* `rwlock_acquire_writelock()`: acquires a write lock
		* `rwlock_release_writelock()`: releases it
		* use a semaphore to ensure only single writer acquires lock and enters critical section
	* `rwlock_acquire_readlock()`: write lock by calling sem_wait() on writelock semaphore and releasing lock by calling `sem_post()`occurs when first reader acquires the lock
	* ```c
	  void rwlock_acquire_readlock(rwlock_t *rw) {
		  sem_wait(&rw->lock);
		  rw->readers++;
		  if (rw->readers == 1) {sem_wait(&rw->writelock());} //first reader gets the write lock, since there's only one of these, any other writer wanting to get a lock doesn't get it because it would cause semaphore to go negative.
		  sem_post(&rw->lock);
	  }
	  
	  ```
* dining philosophers problem:
	* need to get forks and put forks such that there's no deadlock no philosopher starves and never gets to eat, and concurrency is high
	* helper functions to get left and right fork:
		* ```c
		  int left(int p) {return p;}
		  int right(int p) {return (p + 1) % 5;}
		  ```
	* simple approach:
		* `sem_wait(&forks[left(p)]` and same thing w/ right for getting forks
		* `sem_post(&forks[left(p)])` and same thing w/ right for putting forks
		* issue w/ this:
			* dead lock, if everyone grabs fork on left other than right, then every one is waiting for another forever
				* philosopher 0 grabs fork 0
				* philosopher 1 grabs fork 1
				* philosopher 2 grabs fork 2
				* philosopher 3 grabs 3...
		* the idea is break the pattern:
			* have the fourth philosopher grab right before left (this way we don't have a situation where everyone is trying to grab one before the other)
			* breaks a cycle
* can throttle the # of threads (called admission control)
	* decide on a threshold and limit # of threads concurrently executing piece of code
	* initialize a semaphore w/ the max number of threads u wish to enter a memory-intensive region, putting a `sem_wait()` and `sem_post()` naturally throttles threads in a critical section
* implementing semaphore:
	* ```c
	  void zem_wait(zem_t *s) {
		  pthread_mutex_lock(&s->lock);
		  while (s->value <= 0) {
			  pthread_cond_wait(&s->cond, &s->lock); //once negative we wait
		  }
		  s->value--;
		  pthread_mutex_unlock(&s->lock);
	  }
	  
	  void zem_post(zem_t *s) {
		  pthread_mutex_lock(&s->lock);
		  s->value++;
		  pthread_cond_signal(&s->cond);
		  pthread_mutex_unlock(&s->lock);
	  }
	  ```