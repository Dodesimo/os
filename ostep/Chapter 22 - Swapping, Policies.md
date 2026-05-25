* memory pressure, OS needs to page out to make room for actively-used pages
* deciding which pages to evict: based on the replacement policy of the OS
* main memory: holds a subset of all pages in system, so its a cache for virtual memory pages
	* minimize # of cache misses: minimize # of times we fetch page from disk
	* or maximize # of cache hits, # of times, page is accessed and found in memory
* Average mmory access time (AMAT): calculated this in 2200
* tiny miss rate: will dominate overall AMAT and cause issues
* belady's min: remove the page that will accessed furthest in the future (optimal policy)
	* throw out page farthest from now
	* all the other pages in the cache are more important than farthest one out
* cold-start miss (compulsory miss): cache is empty
* capacity miss: cache ran out of space, needed to evict an item to bring a new one in
* conflict miss: item already there in terms of associativity but cache has spaces
* FIFO: pages placed in a queue when entering system, when replacement happened, tail evicted
	* does significantly worse than optimal, can't determine the important of blocks
* random: picks a random page to replace under memory pressure
	* simple to implement
	* performance determines on choices made, so average out over thousands of times
* LRU:
	* commonly used property of page: recency of access. (recently accessed, more likely to be accessed again)
		* less frequently used is frequency (page accessed many times, shouldn't be replaced)
	* more recently a page is accessed, more likely it gets accessed again
		* based on the principle of locality, observation about programs/behavior
		* programs tend to access certain code sequences and data structures frequently (use history to figure out which pages are important)
		* keep those pages in memory when it comes to eviction
	* LFU: replaces least-frequently used page
	* LRU: replaces least-recently used page
		* gets near optimal
* with no locality, doesn't matter what eviction policy is used (everything performs the same)
	* hit rate is directly determined by size of cache
	* if cache can fit the entire workload, doesn't matter what policy is used
* with locality (80-20 load, 80% of references made to 20% of pages), LRU does significantly better (since hot pages are better held onto since they've been referred to frequently in the past)
	* matters more if misses are very costly
* looping sequence: refer to 50 pages in sequence, starting from 0 up to page 49
	* LRU and FIFO: kick out older pages, however due to looping, they are accessed sooner than pages policies prefer in cache
* how do we implement historical properties?
	* LRU: lots of work (each page access, update a data structure to move this page to front of list)
	* FIFO: list only accessed when page evicted, or new page added
	* keep track of pages has least and most recently used, account every memory reference
	* machine keeps rack of every page access, time field gets set to the current time
		* replacing a page: OS scans all time fields
	* approximation is better:
		* has hardware support in the form of use bit/reference bit
		* use bits live in memory somewhere
		* page referenced, use bit is set by hardware to 1
	* how to determine what page to evict based on the bits?
		* clock algorithm: start at a particular page, if 1, set use bit to 0, then move to next page (increment), keep going till we get a use bit that's 0.
	* prefer to evict clean pages over dirty ones because dirty ones need to be written back and that's expensive
* when are pages brought into memory? (page selection):
	* demand paging: OS brings page into memory when asked (could predict through a prefetch)
* how does the OS write pages out to disk?
	* systems collect # of pending writes together, write them to disk in one efficient write
* thrashing: OS' memory is oversubscribed, memory demands of running process exceeds available physical memory
	* situation called thrashing
* admission control: given a subset of processes, system chooses not to run a subset so that reduced working sets fit in memory
	* better to do less work well
* out of memory killer: when memory is oversubscribed, memory-intensive process gets killed
	* problems if the thing getting killed is a server for example that's in active use
* scan resistance: new types of algorithms that prevent worst case for LRU where we loop
* 