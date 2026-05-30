- cases where thread wishes to check whether condition is true before continuing execution
- parent wants to know when child is done
- can be done w/ a shared variable
	- requires spinning and wastes CPU time, also not efficient with many children spawned in
- use condition variable: explicit queue threads can put themselves on when some state of execution isn't met
	- waiting on the condition
	- some other thread, when it changes state, can wake one or more of these threads and allow them to continue
- `pthread_cond_t c`: two operations `wait()` and `signal()`
	- `wait()`: thread wishes to put itself to sleep
		- `pthread_cond_wait(pthread_cond_t *c, pthread_mutex_t *m)`
		- takes in a conditional variable and a mutex
		- the mutex is assumed to be locked, releases the lock, puts calling thread to sleep
		- when thread wakes up, re-acquires lock before returning to caller
	- `signal()`: thread changed something and wants to wake up a sleeping thread waiting on this condition
		- `pthread_cond_signal(pthread_cond_t *c)`
	- wait example
		- ```c
		  void thr_exit() {
			  pthread_mutex_lock(&m);
			  done = 1;
			  pthread_cond_signal(&c);
			  pthread_mutex_unlock(&m); //when child is done, awakens parent thread that slept on conditional variable c
		  }
		  
		  void thr_join() {
			  pthread_mutex_lock(&m);
			  while (done == 0) {
				  pthread_cond_wait(&c, &m); //while the child isn't done, we sleep, adding our self to the condition variable and wait lets go of our mutex 
			  }
			  pthread_mutex_unlock(&m);
		  }
		  
		  ```
	* parent: if it keeps running, goes to sleep, child runs, signals parent thread, parent runs w/ lock, unlock lock and is done
	* state variables mandatory:
		* otherwise won't wake up
	* no lock:
		* ```c
		  void thr_exit() {
			  done = 1;
			  pthread_cond_signal(&c);
		  }
		  
		  void thr_join() {
			  if (done == 0) {
				  pthread_cond_wait(&c);
			  
		  }
		  ```
		* race condition: if parent gets interrupted before sleeping, child runs and signals (but no one there), and then parent goes to sleep and no one wakes it 
	* hold the lock when calling signal or wait
* producer/consumer problem:
	* producer puts items on a buffer, consumer grabs items from buffer and consumes them
	* used in multi-threaded web servers: producer puts HTTP requests into work queue, consumer takes requests out and processes them
	* also used for pipes: `grep foo file.txt | wc -l`:  
		* output of first part is connected to the input of the second part
		* `grep` process is the producer, `wc` process is the consumer
	* shared resource, so we need a synchronization primitive 
	* data should be in buffer when count is 0
	* only get data from buffer when count is 1
	* issue w/ using `if` for buffer size checks:
		* only preserves the initial condition
		* producer produces, signals to consumer, consumer is ready to run, some other consumer gets scheduled, consumes the item, first consumer encounters a queue size of 0
			* issue: after producer woke up first consumer but before it ran, the state of the bounded buffer changed
			* no guarantee that when the woken thread runs the state will be the same
				* known as Mesa semantics (waiting thread is transitioned back to the ready but doesn't run immediately, other threads might run first, need to validate condition)
				* Hoare semantics: stronger, immediately runs
	* issue w/ having just a single condition variable:
		* two consumers go to sleep because queue empty, producer produces, wakes up one consumer, then producer goes to sleep
		* consumer consumes, sees its empty, needs to signal producer but since both producer and consumer share the same conditional variable, can't do so
		* need to have an empty condition variable and a full condition variable
		* ```c
		  void *producer (void *arg) {
			  int i;
			  for (i = 0; i < loops; i++) {
				  pthread_mutex_lock(&mutex);
				  while (count == MAX) {
					  pthread_cond_wait(&empty, &mutex); // if full wait till empty
				  }
				  put(i);
				  pthread_cond_signal(&fill); //tell consumers the queue is ready to be taken from
				  pthread_mutex_unlock(&mutex);
			  }
		  }
		  
		  void *consumer (void *arg) {
			  int i;
			  for (i = 0; i < loops; i++) {
				  pthread_mutex_lock(&mutex);
				  while (count == 0) {
					  //we want to wait till we are empty
					  pthread_cond_wait(&fill, &mutex);
				  }
				  int temp = get();
				  pthread_cond_signal(&empty); //tell threads waiting that its empty
				  pthread_mutex_unlock(&mutex);
			  }
		  }
		  ```
	- use while especially to check for spurious wake-ups
	- for a correct solution, just add more slots to producer and consumer
		- this allows for multiple values to be consumed before sleeping
		- multiple producers/consumers: allows concurrent producing/consuming to take place
	- memory allocation:
		- when we free memory, and we signal to threads that memory is available, how do we know what threads to signal?
			- solution: use `pthread_cond_broadcast()`: will wake up all threads, each thread will see whether the provided back memory is enough `(bytesLeft < size)`
				- sleep if needed
				- negative performance impact, might needlessly wake up many unnecessary threads
	