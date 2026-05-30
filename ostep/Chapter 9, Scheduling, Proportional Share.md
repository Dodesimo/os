- Scheduler tries to guarantee every job a certain percentage of CPU time.
	- Lottery scheduling: hold a lottery to determine which process should run next, processes running more often should be given more chances.
- Lottery scheduling: tickets, used to represent share of resource that process receives.
	- So percent of tickets that a process has: represents its share of system resource.
	- If we want process A to run on CPU 75% of the time, and process B to run on CPU 25% of the time, have a 100 tickets, and 75 tickets for process A, and 25 tickets for process B.
- Randomly pick a ticket every time slice.
	- Randomness: avoids strange corner-case behaviors (such as an LRU for example), lightweight (has little state needed to track), random can be fast (just need to generate a random number).
- Randomness: leads to probabilistic correctness, but no guarantee.
	- Longer jobs complete, more likely they achieve desired percentages due to CLT.
- Use tickets to represent a proportion of ownership.
- Different ticket mechanisms:
	- Ticket currency: user w/ set of tickets can allocate amongst own jobs in whatever currency.
		- This gets converted to the correct global value.
	- Ticket transfer: process temporarily hands off tickets to another process (useful in a client/server setting).
		- Client process sends message to server asking to do work on its behalf.
			- Give the server its tickets to maximize its performance (server gets more CPU share to do work).
	- Ticket inflation: boost # of tickets, all processes trust one and another so if one process knows it needs more CPU time (boost ticket value).
- Implementation:
	-  Generate a random number.
	- Go through a process list, and sum up the total number of tickets.
		- If > random number, break, we found our process.
	- This works because the number of tickets that each process has represents a range.
	- ```c
	  int counter = 0;
	  node_t * current = head; 
	  while (current) {
		  counter += current->tickets;
		  if (counter > randomNumber) {
			  break;
		  }
		  current = current->next;
	  }
	  ```
	* Efficiency purposes: sort greatest to smallest (least amount of list iterations done that way).
* Fairness metric: time first job completes divided by time second job completes (want to be 1).
	* Fairness can be low when job length is not that low.
		* Only when jobs run for significant time slices do we have a desired fair outcome.
* How do we assign tickets?
	* This is an open problem.
	* Can just hand them to user and tell them to assign to jobs but doesn't tell us what to do.
* Stride scheduling:
	* For short time-scales, this doesn't get us right proportions.
	* Each job has a stride, inverse in proportion to # of tickets it has.
		* Divide some large # by # of tickets :
			* If A, B, and C have 100, 50, 250 tickets, divide 10,000 by these ticket values and get stride values.
			* Use this stride to increment each process' counter.
			* To pick a process to run, use the one with the lowest pass value.
	* Stride scheduling achieves right proportions of scheduling at the end of each cycle, lottery scheduling achieves proportions probabilistically over time
	* Why lottery better?
		* No global state.
		* If new job enters, what should its pass value be (can't be 0 because then it monopolizes CPU).
* Linux Completely Fair Scheduler (CFS):
	* Want to reduce overhead of  scheduling decisions (through design and use of data structures)
	* Fairly divide a CPU among all competing processes.
		* Counting technique called `virtual runtime`.
		* As each process runs, accumulates `vruntime`, which increases at same rate as the physical time.
		* Pick process w/ lowest `vruntime`.
	* How does scheduler know when to stop and run?
		* Switch too often, fairness is increased.
		* But if switches less often, performance is increased but not fair.
	* Use `sched_latency`:
		* Determines how long a process should run before a switch (typically its value is 48 millseconds)
		* Divides this value by the number of processes to determine time slice of a process.
	* Too many processes: would cause small time slices, resulting in too many context switches.
	* Add another param: `min_granularity`:
		* never set a time slice below this.
		* Won't completely be fair, but would achieve high CPU efficiency.
	* CFS: has a periodic timer interrupt so can only make decisions at fixed time intervals.
		* Determines every 1 ms (every time timer goes off) to see if current job is end of run.
		* If time slice isn't perfect multiple of interrupt interval, ok b/c CFS tracks `vruntime` precisely.
			* So we will approximate ideal sharing of CPU.
	* How to give processes priority?
		* use a `nice` level: set from -20 to +19 (negative values mean greater priority, positive values mean less priority)
		* maps to a series of weights, these weights are used to weight the time slice
			* $timeslice_k = \frac{weight_k}{sum of all weights} * scheduled_latency$
	* vruntime: vruntime + initialweight/priority weight * runtime
	* CPU proportionality ratios are preserved when difference in nice values is constant.
	* How to efficiently get the lowest `vruntime`:
		* red-black tree, balanced tree so we have low depth to search (`O(log n)` search times instead of degenerating to `O(n)`).
		* Not all processes are added to this structure; only runnable ones.
	* If process B has slept for a while, its `vruntime` is behind A for a while and could monopolize the CPU (starving A).
		* When a process wakes up, it makes its vruntime the minimum in the tree.
			* Since this is the minimum time a running process has, the new process cannot monopolize the CPU.
			* Prevents starvation: bust cost is jobs that sleep for short periods of times don't get a fair CPU share.
	* Ultimately: CFS is a weighted round-robin ith dynamic time slices.