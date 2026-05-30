* How do you allocate/manage memory?
* Stack memory: allocations/deallocations are managed implicitly by the compiler.
	* Called automatic memory.
* Returning from a function, deallocates memory.
* Heap memory: allocations/deallocations explicitly handled by the programmer.
	* ```c++
	  void func() {
		  int *x = (int *) malloc(sizeof(int));
	  }
	  ```
	* Both stack and heap allocation goes on here.
		* Stack: make room for a pointer to an integer.
		* When program calls `malloc()`: requests space for an integer on the heap.
			* Routine returns address of integer stored on the stack for use by the program.
* `malloc()`: pass in a size, and either succeeds an returns back a pointer to newly-allocated space or returns NULL.
* ```c++
  double *d = (double *) malloc (sizeof(double));
  ```
- compile-time operator since we know the actual size during compilation (a number is substituted to the argument.)
	- `sizeof()`: operator not a function call.
- `sizeof` can be incorrect at times:
	- ```c++
	  int *x = malloc(10 * sizeof(int));
	  printf("%d\n", sizeof(x)); // returns the size of the pointer not the array.
	  ```
* can be correct:
	* ```c++
	  int x[10];
	  printf("%d\n", sizeof(x)); //correctly prints size of entire array.
	  ```
* Allocating space for strings:
	* `malloc(strlen(s) + 1)` -> accounts for null character.
* `malloc()`: returns a pointer to type void.
* `free()`:
	* takes in one argument, pointer returned by `malloc()`.
	* Doesn't keep track of the portion of memory this free pointer accesses (tracked by the memory allocation library itself).
* Automatic memory management: garbage collection runs and figures out what memory is no longer referenced, and frees it.
* Some methods anticipate preallocated memory associated with pointers:
	* ```c++
	  char *src = "hello";
	  char *dst; // expects this to be a prealocated pointer.
	  //correct
	  char *dst = (char *) malloc(strlen(src) + 1);
	  strcpy(dst, src);
	  ```
- Not allocating enough memory: buffer overflow.
	- Even though a program ran correctly once, doesn't make it correct.
- With `malloc()`, initialize the data (even though it might be init to empty initially).
	- If not init, program may encounter uninit. read where it reads from heap data unknown value.
- Also don't forget to free memory.
	- Causes memory leaks (long-running applications, can cause a run-out of memory).
- For small programs, maybe that when program exists, the OS cleans up everything.
	- But this is a bad habit.
	- This is because there's two levels of memory management.
		- OS: hands out memory processes when they run.
		- Second level: within each process when `malloc()` and `free()`.
		- if not free, OS claims everything for the process including pages for code stack and heap.
- Freeing memory before all operations are complete results in operations on a dangling pointer, results in crashes.
	- But don't double free at the same time.
	- Free needs a `malloc()` returned pointer.
- Underlying OS calls for `malloc()` and `free()`.
	- They are library calls built on top of system calls.
	- one such call: `brk`: change location of the end of the heap.
		- Takes an argument: address of new break.
		- Either increases or decreases heap size based on it.
	- `sbrk`: passes an increment.
		- returns prior value of the program break.
	- `brk`: returns 0 if succcessful.
	- If either not successful, return a -1, set the `errno` global variable.
- `mmap()`: creates anonymous memory region that isn't associated with a file but swap space.
	- Can also be treated as a heap.
- `calloc()`: allocates memory, also zeroes it out.
- `realloc()`: makes a larger region of memory, copies old region into it, and returns pointer to the new region.
- 