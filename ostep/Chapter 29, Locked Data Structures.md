- how to add locks to data structures for functionality and optimal concurrency
- counter:
	- follows basic design pattern
	- adds single lock, acquired when calling routine, released when returning from the call
	- similar to monitor: locks acquired and released automatically as call/return from object methods
	- ```c++
	  typedef struct __counter_t {
		  int value;
		  pthread_mutex_t lock;
	  } counter_t;
	  
	  void init (counter_t) {
		  c->value = 0;
		  pthread_mutex_init(&c->lock, NULL);
	  }
	  
	  void increment(counter_t *c) {
		  pthread_mutex_lock(&lock);
		  c->value++;
		  pthread_mutex_unlock(&lock);
	  } //similar pattern follows for all other methods in terms of locking
	  ```
	* scaling is bad w/ this: more threads increase time needed to increment
		* more time spent on lock acquisition and such
	* perfect scaling: more work done, done in parallel so time taken isn't increased
*  approximate counter:
	* local physical counters exist per CPU core
	* single global counter, four local counters
	* locks: one for each local counter, one for global counter
	* thread running on given core increments counter, does it on local w/ lock
		* still need a lock because thread could get preempted and increment isn't atomic
	* keep a global counter up to date: local values are periodically transferred to global counter
		* gather lock increment local counter's value, local counter is then reset to 0
		* this transfer is determined by a threshold S
			* smaller S, more counter is a non-scalable counter
			* larger S, more scalable counter is, but more inaccurate global count is
* concurrency isn't necessarily faster:
	* overhead of acquisition/release of locks is time consuming
	* simple schemes work well, adding more locks can be an issue
* concurrent linked list:
	* for performance purposes, the lock should be only in the critical section (code where a shared variable is getting accessed)
	* for example:
		* ```c++
		  int List_insert(list_t *L, int key) {
			  node_t *new = malloc(sizeof(node_t));
			  if (new == NULL) { return -1; } //we failed
			  new->key = key;
			  //only need to lock critical section when we manipulate linked list
			  pthread_mutex_lock(&L->lock);
			  new->next = L->head;
			  L->head = new;
			  pthread_mutex_unlock(&L->lock); //-> is higher precedence op than &
			  return 0;
		  }
		  ```
	* better approach: hand over hand locking
		* add lock a for each node of the list
		* traversing the list, code grabs next node's lock, releases present node
		* however, with very large lists, large # of threads, concurrency allowed isn't going to be that much faster due to the overhead
* concurrent queue:
	* two locks: one for the head one for the tail
		* tail is for dequeueing
		* head is for enqueueing
	* ```c++
	  void Queue_Enqueue(queue_t *q, int value) {
		  node_t *tmp = malloc(sizeof(node_t));
		  assert(tmp != NULL);
		  tmp->value = value;
		  tmp->next = NULL;
		  
		  //critical section
		  pthread_mutex_lock(&q->tail_lock);
		  q->tail->next = tmp;
		  q->tail = tmp;
		  pthread_mutex_unlock(&q->tail_lock); // same thing applicable to queue dequeue
	  }
	  ```
* concurrent hash table:
	* can just have an array of linked lists that are concurrent
	* has high performance because the entire hash table isn't required to be locked, only lists associated with particular keys
		* so multiple key lists can be processed
		* essentially a lock per hash bucket
	* ```c++
	  int Hash_Insert(hash_t *H, int key) {
		  return List_Insert(&H->lists[key % BUCKETS)], key);
	  }
	  ```