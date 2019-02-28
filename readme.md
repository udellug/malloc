# malloc, or how I learned to stop debugging with printf

## pointers

Pointers are special variables that aren't *really* variables per se

They only show up in languages with manual memory management
(C, C++, Ada, Go, etc.) because they're used to manually manage memory
in ways that Java and Python don't allow.

Pointers refer to the location of a variable in memory. The type of
pointer specifies what is at that location. You can think of pointers
as addresses (which they are, for the most part). My address tells you
where to find my house, but it isn't my actual house. Going there and
changing what my house looks like (by painting the walls or mowing the
lawn) is something you have to do by yourself.

In C pointers are declared using the `*` symbol.

```c
int *pc; // this declares a pointer that holds the address of an int
int c; // this declares an int
int b,*pb;
/*
 * you can also declare pointers with comma notation alongside variables
 *  of the same type as the pointer
 *
 * note that an int * is not a different type than an int. this is weird
 *  but cool
 */

c = 10; // this changes the value of c
pc = &c;
// this sets pc to point to c. & is the "reference operator"
*pc = 6; // this changes the value of the object at pc
// * is the dereference operator (this is confusing but fine)

/*
 * c == 6
 */

```

## the stack vs the heap

### local vars vs dynamic memory

Most of the time when you want a variable you just instantiate it
but you don't really care where the computer stores it. This is fine,
but not always what we want.

Any variable declared in a normal function goes on the stack,
which is local to that function. Once the function exits, the variable
is deleted and the computer lets other things use that space in memory.

If we want to keep things that are too big to pass around on the stack
(think big structs, objects if you're into C++, that kind of thing),
we stick them on a special place in memory called the heap, which
doesn't automatically delete things.

This is called dynamic memory because it's allocated dynamically at
runtime.

## malloc

`malloc()` is the function that dynamically allocates memory in C.

```c
void *p;
p = malloc(41); // p now points to 41 bytes of memory on the heap
```

`malloc()` has a very important partner, `free()`.

```
void *p;
p = malloc(30283); // we need a LOT of space for something
...
free(p); // make sure we give it back though
```

Free as in malloc is also a good blog that you should read.

There are other mallocs, which do cool things like:

  * make sure the memory you ask for doesn't have garbage data in it
  * make sure the memory you ask for does have garbage in it, but garbage
  that you specifically ask for
  * take your memory and give you a bigger memory, because you asked
  for the wrong size

## malloc implementation

### how to get more space

Your program has different sections for different types of memory:

  * the actual code (.text)
  * global variables and static variables (.data)
  * uninitialized data (.bss)
  * the stack & heap
  * commandline args, environment variables, etc

When loaded into memory, it looks like this:

```
+------------------+ 0xffffffff
|                  |
| commandline args |
| environment vars |
| etc.             |
|                  |
+------------------+
|                  |
| stack            |
|                  |
+------------------+
|        |         |
|        |         |
|        v         |
|                  |
|                  |
|        ^         |
|        |         |
|        |         |
+------------------+
|                  |
| heap             |
|                  |
+------------------+
|                  |
| .bss             |
|                  |
+------------------+
|                  |
| .data            |
|                  |
+------------------+
|                  |
| .text            |
|                  |
+------------------+ 0x00000000

```

There's a cool syscall called `brk` that moves the end of the .bss
section to whatever address you give it. Under the hood, `malloc()`
uses this to increase the .bss section size to fit any dynamic memory
you might need

```c
struct block_meta *request_space(struct block_meta *last, size_t size){
  struct block_meta *block;
  block = sbrk(0); // current program break
  void *request = sbrk(size + META_SIZE);
  assert((void*)block == request); // not thread safe
  if(request == (void*) -1){
    return NULL;
  }

  if(last) { // NULL on first request
    last->next = block;
  }
  block->size = size;
  block->next = NULL;
  block->free = 0;
  block->magic = 0x12345678;
  return block;
}

```

### what to do with old space

Unlike the stack, we don't know when memory is going to be freed,
so we can't just move the program break (the end of the .bss segment)
back and forth willy-nilly, otherwise we risk abandoning some memory
that's still in use

Instead, as you might have gathered from the code demonstrating
`sbrk()` to increase the .bss, dynamic memory is stored as a linked
list, with metadata at the beginning that looks like this:

```c
struct block_meta{
  size_t size;
  struct block_meta *next;
  int free;
  int magic; // debugging only TODO: remove this
};
```

When we free a block of dynamic memory, all we have to do is set the
`free` field to 1 (or a nonzero value).

```c
void free(void *ptr){
  if (!ptr){
    return;
  }

  // TODO: consider merging blocks once splitting blocks is implemented
  struct block_meta* block_ptr = get_block_ptr(ptr);
  assert(block_ptr->free == 0);
  assert(block_ptr->magic == 0x77777777 || block_ptr->magic == 0x12345678);
  block_ptr->free = 1;
  block_ptr->magic = 0x55555555;
}
```

### actual malloc internals

```c
void *malloc(size_t size){
  struct block_meta *block; // TODO: align size?

  if (size <= 0) {
    return NULL;
  }

  if (!global_base) { // first call
    block = request_space(NULL, size);
    if (!block) {
      return NULL;
    }
    global_base = block;

  } else {
    struct block_meta *last = global_base;
    block = find_free_block(&last, size); // traverse the heap

    if (!block) {
      return NULL;

    } else { // found free block
      // TODO: consider splitting free block here
      block->free = 0;
      block->magic = 0x77777777;
    }
  }
  return (block+1); // return pointer to space after metadata block
}
```

demo/story about debugging

memes
