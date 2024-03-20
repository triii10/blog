---
title: "C Memory Management Rules"
date: 2024-03-20T14:48:31Z
draft: false
toc: true
images:
tags:
  - c
  - c++
  - secure coding
---


# Reference: [SEI CERT C Coding Standard](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard)


### MEM30-C (Do not access freed memory)

The following behaviours relating to the use of pointers, after they have been deallocated, are **undefined** :

- Evaluating a pointer, including dereferencing it
- Using it as an operand of an arithmetic operation
- Type-casting it
- Using it as the right-hand side assignment

Such pointers are called *dangling pointers*. Accessing such pointers can lead to exploitable vulnerabilities.

##### Example 1 - 

```c
#include <stdlib.h>
struct node { 
	int value; 
	struct node *next; 
}; 
void free_list(struct node *head) { 
	for (struct node *p = head; p != NULL; p = p->next) {
		free(p); 
	} 
}
```
In this example, `p->next` is accessed after `p` is already freed.

##### Example 2 - 

```C
#include <stdlib.h> 
#include <string.h> 
int main(int argc, char *argv[]) { 
	char *return_val = 0; 
	const size_t bufsize = strlen(argv[0]) + 1; 
	char *buf = (char *)malloc(bufsize); 
	if (!buf) { 
		return EXIT_FAILURE; 
	}
	/* ... */ 
	free(buf); 
	/* ... */ 
	strcpy(buf, argv[0]); 
	/* ... */ 
	return EXIT_SUCCESS; 
}
```

In above code snippet, `buf` is being written with the value of `argv[0]` after it has been freed.

##### Example 3 - 

```C
#include <stdlib.h> 
void f(char *c_str1, size_t size) { 
	char *c_str2 = (char *)realloc(c_str1, size); 
	if (c_str2 == NULL) { 
		free(c_str1); 
	} 
}
```

In the example above, `realloc` may free `c_str1` when it returns a null pointer, resulting in `c_str1` freed twice. Freeing a pointer twice can result in *double-free vulnerability*.

### MEM31-C (Free dynamically allocated memory when no longer needed)

Before the lifetime of the last pointer (that stores the return value of a call to a standard memory allocation function) has ended, it must be matched by a call to `free()`.

##### Example 1

```C
#include <stdlib.h> 
enum { BUFFER_SIZE = 32 }; 
int f(void) { 
	char *text_buffer = (char *)malloc(BUFFER_SIZE); 
	if (text_buffer == NULL) { 
		return -1; 
	}
	return 0; 
}
```
In above example, `text_buffer` is not freed before the function returns.

##### Exception - a static pointer does not needed to be freed.

### MEM33-C (Allocate and copy structures containing a flexible array member dynamically)

Below example - 

```C
struct flex_array_struct { 
	int num; 
	int data[]; 
};
```

The size of the struct `flex_array_struct` is determined by only the first member `num`, since the size of the array `data` is not fixed.
It is undefined behaviour to access the `data` member from an instance of the struct, as no space is reserved for the `data` member on stack.

Following should be adhered to avoid potential for undefined behaviour - 
1. Have dynamic storage duration for the flexible array structure (like using malloc())
2. Be dynamically copied using memcpy(), or something similar, but not by assignment
3. When used as an argument to a function, be passed by pointer and no copied by value

### MEM34-C (Only free memory allocated dynamically)

Freeing memory that is not allocated dynamically can result in heap corruption and other errors.
Do not call `free()` on a pointer other than the one returned by a standard memory allocation function such as `malloc()`, `calloc()`, etc.

### MEM35-C (Allocate sufficient memory for an object)

The types of integer expressions used as size arguments to memory allocation functions must have sufficient range to represent the size of the objects to be stored.

```C
#include <stdint.h> 
#include <stdlib.h> 
void function(size_t len) { 
	long *p; 
	if (len == 0 || len > SIZE_MAX / sizeof(*p)) { 
		/* Handle overflow */ 
	}
	p = (long *)malloc(len * sizeof(int)); 
	if (p == NULL) { 
		/* Handle error */ 
	}
	free(p); 
}
```

In the above example, `p` is allocated with size of `len * sizeof(int)` which is not sufficient. Compliant solution can be `len * sizeof(long)` or `len * sizeof(*p)`

### MEM36-C (Do not modify the alignment of objects by calling `realloc()`)

Do not invoke `realloc()` to modify the size of allocated objects that have stricter alignment requirements than those guaranteed by `malloc()` (eg, `assigned_alloc()`)