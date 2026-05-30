- When is free-space management harder?
	- When we have variable-sized units (such as in user-level memory libraries like `malloc()`).
	- Or when using segmentation for virtual memory.
- external fragmentation: free space is chopped into little pieces of different sizes, fragmented.
	- even if free space exceeds request size, not enough contiguous space.
- `void free(void *ptr)`: when freeing memory: user does not provide information about how large the space is.
	- library needs to keep track of how large these chunks are.
- free space: heap free-list.
	- contains reference to all free-spaces in managed memory.
- compaction: viable solution, but would be costly for the OS.
- free list: request for anything greater than the smallest element fails.
	- nothing else is contiguous.
- request made that's smaller than both free blocks:
	- split the second, return the portion to the user.
	- remainder is kept in the free list.
- coalescing: returning a free chunk in memory, look at chunk and nearby chunks 
	- if they are adjacent, merge them.
	- this ensures that large extents of memory is available for the application.
- how to track size of allocated regions:
	- header block: kept in memory
	- contains the size of allocated region
	- additional pointers to help w/ deallocation
	- magic #: provide integrity checking
- when`free(ptr)` called:
	- ```c
	  void free(void * ptr) {
		header_t *htpr = (header_t) ptr - 1; //move to the beginnning of the header
	  }
	  ```
	* check whether magic number matches expected value
	* calculate total size of the newly-freed region by adding header size to region
	* when user asks for N bytes of memory, doesn't search for N, but free chunk of N + header.
* within the memory allocation library: need to build the free list.
* heap is build within free space through system call like `mmap()`:
	* when free is called, split to size of chunk + header size, that becomes its own node (existing node - chunk - header_size)
	* pointer returned to right after the header
	* Free list is just a single node pointed to by head.
		* when chunk is freed and returned, we add it to head of the free list.
	* so we start off with a head thats the whole free address space.
		* gets split into taken chunks
		* taken chunks then when freed then point to the head.
	* when everything gets freed: need to coalesce the list.
* growing the heap:
	* if heap runs out of space, fail and just return NULL.
	* or OS uses `sbrk`, grows the heap, allocates new chunks
		* servicing this: free physical pages get mapped to address space of requesting process
	* best fit: find smallest fit that can be found through a pass.
		* massive performance penalty when doing exhaustive search for correct free block
	* worst fit: largest chunk, return requested amount, keep the remaining chunk on the free list
		* so do splits
		*  can lead to excess fragmentation. 
	* first fit:
		* give first block big enough and return requested amount to the user
		* fast, doesn't exhaustively search free space, pollutes beginning of free list with small objects.
			* address-based ordering: keeping list ordered by address of the free space, colaescing is easier. 
			* if we sort our free list by address we can easily merge.
	* next fit:
		* keeps extra pointer to location within list where we looked last.
			* then start searches there.
		* spreads searches for free space more uniformly.
			* avoids splintering of the list beginning.
	* segregated list:
		* application has one or few popular sized requests, keep separate list just to manage objects of that size.
		* fragmentation is less of a concern with similar sized requests coming in.
			* allocation/free requests can be served quicker.
		* issues:
			* how much memory should be in this pool.
	* kernel: allocates object caches for kernel objects requested frequently
		* locks, inodes
		* these are segregated free lists of a given size (serve requests quickly)
		* if a list is running low on space, requests slabs from a more general memory allocator
		* if all reference counts of objects in a given slab are 0, general allocator can reclaim the allocator.
		* free objects on the lists are preinit. (because init and destructor are expensive).
	* binary buddy allocator:
		* we only give out power of two sized blocks.
		* recursively find this in a free space
			* divide it by two till we get a block large enough
		* when freed, look at "buddy" block and if its free, we coalesce the two.
		* easy because address of each buddy pair differs by a single bit at the level (since there are two options at each level)
	* other possibilities:
		* lack of scale (free list is organized in a balanced tree or other data structure).
		* 