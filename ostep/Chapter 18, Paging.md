* Segmentation: dividing space into different-sized chunks is fragmenting and allocation can be hard.
* Paging: chunk up the space into fixed-sized pieces.
	* address space is divided into fixed-size units
* Physical memory: array of fixed sized slots called page frames
	* Each frame contains a single virtual-memory page.
* Benefits of paging: flexibility, system can support abstraction of address space effectively regardless of how process uses address space
	* simplicity: find 4 free pages, just get first pages off of the free list.
* page table: per process data structure that records where each virtual page of address space is in physical memory
	* Lets us know where in physical memory each virtual page resides
* Virtual address: split into virtual page number and offset in the page
	* Use virtual page number to then index into page table and find what physical frame page resides in.
* Page tables are large given the number of pages that may exist.
	* They are stored in memory.
	* Typically at the first page frame of physical memory.
* Page table organization:
	* simples form is a linear page table array, OS indexes the array by the virtual page number and looks up the PTE to find PFN
	* valid bit: indicates whether particular translation is valid
		* used to mark unused pages in address space invalid.
	* protection bit: tells us whether page can be read from, written to, executed from
	* present bit: page is in physical memory or disk (swapped out)
	* dirty bit: indicates whether page has been modified since brought into memory.
	* reference bit: track if page is accessed (useful for page replacement).
* how to do find the page table:
	* page table base register: contains physical address of the starting location
	* ```
	  VPN = (VIRTUALADDRESS & VPN_MASK) >> SHIFT (and gets the bits of the virtual address, and then we shift them to the right to get them in the right magnitude)
	  PTEAddr = PageTableBaseRegister + (VPN * sizeof(PTE)) (VPN is what position to index in, so multiply by the size of the PTE to get how many PTEs past the start of the PageTableBaseReigster we should access)
	  ```
	* ```
	  offset = VirtualAddress & OFFSET_MASK (get the offset from the virtual address)
	  PhysAddr = (PFN << SHIFT) | offset (move the pfn to make space for the offset then set the lower offset bits by this)
 	  ```
 - Every translation of virtual address to physical address requires an extra memory access to page table.
- Each instruction fetch generates two memory references:
	- one to the page table to find physical frame
	- one to the instruction itself to fetch it to the CPU for processing
- `mov`: explicit memory reference (adds another page table access)
	- add a page table access first to get the physical disk array is located on
	- then, access array itself.
- so for the loop: 
	- ```c++
	  for (int i = 0; i < 1000; ++i) {
	  array[i] = 0;
	  }
	  ```
	- for every single iteration:
		- 4 instruction fetches (getting array index and moving 0 to it, incrementing loop var, comparing loop var to end, and then jumping back)
		- Moving to 0 is an additional memory access (explicit memory update)
		- Each of these operations has a page table access.
		- So total 10.