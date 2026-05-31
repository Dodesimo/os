- event-based concurrency
	- used for internet servers, server-side frameworks, GUI-based applications
	- wait for an event to occur, check for event, do small amount of work is needs (like I/O requests, scheduling other events for future handling, etc.)
	- based on an event loop:
		- ```c
		  while (1) {
			  events = getEvents();
			  for (e in events) {
				  processEvent(e);
			  }
		  }
		  ```
		* code that processes each event: event handler
			* when handler processes an event: only activity taking place in the system
			* deciding what event to handle next: scheduling
* how to receive events?
	* basic API through `select()` or `poll()`
		* `select()`: lets us check what descriptors can be read from and written to
			* former: tells us packets have arrived
			* latter: tells us when its ok to reply (out bound queue isn't full)
			* has a timeout: can set to `NULL`: but results in indefinite blocking, better: set it some type of timeout
			* ```c
			  fd_set readFDs;
			  FD_ZERO(&readFDs);
			  int fd;
			  for (fd = minFD; fd < maxFD; fd++) {
				  FD_SET(fd, &readFDs);
			  } //set all interested FDs
			  //select 
			  int rc = select(maxFD + 1, &readFDs, NULL, NULL, NULL);
			  int fd;
			  for (fd = minFD, fd < maxFD, fd++) {
				  if (FD_ISSET(fd, &readFDs)) {
					  processFD(fd); //if its set and we can read from it, process
				  }
			  }
			  
			  ```
* blocking interface: do all work before returning to caller, non-block asynchronous interfaces do some work but return (work gets done in the background)
	* culprit in blocking calls: I/O of some kind
	* non-blocking interfaces: used in any style of programming but essential in event-based approaches (as a call that blocks halts all progress)
* no locks needed: one event gets handled and that's the only thing that gets handled
	* prevents concurrency bugs common in threaded programs
* what if an event requires a system call that might block?
	* no other threads to run other than main event loop
	* implies that if event handler issues blocking call, entire server blocks till completed call
	* no blocking calls are allowed
	* overcome this through async IO
		* enable application to issue I/O request and return immediately to the caller
		* fill out an AIO control block and pass this into asynchronous read API
			* ```c
			  struct aiocb {
				int aio_fildes; //file descriptor
				off_t aio_offset; //file offset
				volatile void *aoi_buf; //location of buffer
				size_t aio_nbytes; //length of transfer
			  }
			  
			  int aio_read(struct aiocb *aiocbp);
			  ```
		* to see if request has completed, continuously poll `int aio_error(const struct aiocb *aiocbp)`
			* if success; return 0 else return `EINPROGRESS`
		* better than polling: interrupts, method uses UNIX signals to tell applications aync I/O is complete
* manual stack management:
	* event handler issues async I/O: package up some program state for next event handler to use when I/O completes
	* use continuations:
		* record needed info to finish processing event in data structure
		* when event happens (such as a disk I/O read) look up info in data structure
		* reading from file descriptor, write to network socket descriptor:
			* use the SD as a key in a hash table indexed by FD
			* disk I/O completes, event handler uses the file descriptor to look up continuation
			* and then returns value to caller 
* issues w/ events:
	* w/ multiple CPUs, parallel event handlers so need locks and other synchronizations mechanism
	* cannot handle implicit blocking (what if an event handler page faults?, server is blocked)
	* nonuniform: need `select()` for networking events and AIO for file I/O