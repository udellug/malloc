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

what it does

free as in malloc joke

different flavors

## malloc implementation

how to get more space

what to do with old space

demo/story about debugging

memes
