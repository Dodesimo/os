- Page tables are large if linear structures.
- Simple solution: bigger pages.
	- 32 bit address bit (12 bits dedicated to page offset, 20 bits dedicated to virtual page number).
	- If we make 14 bit offset, 18 bits for page number, and assuming 4-byte page table entries, total size of 1 MB per table.
	- This causes internal fragmentation (since there's waste in each page).
- For small programs, most of page table is unused and has invalid entries.
- Hybrid approach: have a page table for each logical segment
	- Base + bounds: have differing roles now
	- Base: physical address of page table
	- Bounds: end of the page table
	- Assume there's three base/bounds pair (code, heap, stack) -> thus there's three page tables as well for each process
	- Conversion:
		- ```
		  SegmentNumber = (VirtualAddress & SEG_MASK) >> SN_SHIFT //get the right segment
		  VPN = (VirtualAddress & VPN_MASK) >> VPN_SHIFT // get the virtual number
		  AddressOfPTE = Base[SN] + (VPN * sizeof(PTE)) // then calculate index for the right segment's page table to get the physical frame number
		  ```
	* Base and bounds: causes a lot of memory savings compared to linear page table (unallocated pages between and heap aren't taking up space in page table, they are invalid).
	* Issues: external fragmentation, page tables can now be of arbitrary size (multiples of PTEs).
* Multi-level Page Table:
	* Convert the linear page table into a tree.
	* Chop up page table into page-sized units.
	* If entire page of entires not valid, don't allocate page table at all.
	* Use page directory to determine whether page of page table is valid.
		* Or if entire page table has no valid pages.
	* Difference to classic linear page table: enough though most middle regions not valid, require space for them.
	* Directory: tells us what physical frames worth of page table entries are completely empty and which ones are populated
	* Page directory entries: has valid bit, page frame number
		* If true, means at least one of the pages in the entry 
	* Benefits: only allocates page-table space in proportion to the amount of address space used.
		* Compact and supports sparse address spaces
	* Multilevel structure: add level of indirection through a page directory.
		* Points to pieces of page table, indirection allow us to place whatever pages in there (not necessary contiguous)
	* Disadvantages:
		* TLB miss: two loads from memory are needed to get translation info for page table (one for the page directory, and then one for the appropriate page).
			* time-space trade-off
		* Increased implementation complexity.
* Converting a linear table to a multi-level page table:
	* Gather total size of table.
	* Then divide this size by the size of each physical frame/page.
	* If total size is 1KB, and pages are 64 bytes, table can be split into 16 pages 64-byte pages, each page holds 16 PTEs (since each entry is 4 bytes)
* How does virtual page number work:
	* 16 pages of PTEs: 4 bits
	* Top four bits of the VPN match w/ this (page directory index)
	* Last four bits of the VPN: used to index within a valid page directory to give a page table entry (PTIndex)
* So PDE tell us a PFN that contains PTES for all in-use VPNs.
	* Use top bits of the VPN to index directory
	* then use bottom-top bits to index into the PT portion, get the corresponding PFN, then concatenate the offset to this PFN to get the location in the physical frame.
* Goal: make each portion of the page table fit into a single page frame
* ```
  PTEAddr = (PDE.PFN << SHIFT) + (PTIndex * sizeof(PTE)) // second addition part is the offset, first part is the index
  ```
- Inverted page tables:
	- each page table has an entry for each physical page, tells. us what process is using the page
	- correct entry for a physical page can be found through searching this data structure (but linear scans are expensive, so we use a hash table)
- Can swap page tables to disk when memory pressure gets tight.
	- Some page tables are just too large to fit into memory all at once.
- 