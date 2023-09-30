+++
title = "The Back to Basics Manifesto"
date = "2023-09-27T10:00:00+0000"
author = "Aaron Knudtson"
authorTwitter = "" #do not include @
cover = ""
tags = ["basics", "pointers", "bits", "bytes"]
keywords = ["", ""]
description = ""
showFullContent = false
toc = true
draft = false
+++

# Intro
For too long, CPU speed has been used as an excuse to make slow software. Developers 
are chanting "Programming in C is too slow", "DX&trade;", and "But I just want to code, man!".
Programmers these days also love to complain about how long it takes to ship code 
in any language other than javascript, while at the same time never shipping the 
javascript they are writing.

Taking CPU speed and memory for granted has led to Javascript and Python to take 
the 1 and 2 spots as most used programming language. These are languages where 
"anything goes" - except low memory and fast programs. Sacrificing program speed 
for development speed. Things are not perfect in the land of C and C++ however; althought
npm and pip are not perfect package managers, it is nice to be able to grab a library 
and start using within seconds, and using external libraries is a pain in C and C++.
However, being close to the metal has its advantages - you only use what you need to,
and you can often take basic functions at face value. I don't have to guess what an 
int multiplied by an int is doing, while in Javascript and Python I don't know what 
is acutally happening, other than some probably expensive runtime type checks, operator
function loading, and weird object derefence onion peeling to get to the actual data to finally
carry out a multiplication action. After all, transistors can't get much smaller. I'm not
going to call for an end to Moore's Law, but come on - transistors have reached 5 nm 
in production, and silicon atoms are 0.2 nm in diameter. That's only 25 atoms in between 
transistors. Calculations performed by people much smarter than I show that dropping 
to some single digit amount of atoms make quantum tunneling much more likely, and 
electrons will start to skip across silicon boundaries. Bits will flip out. Not 
to be an alarmist or anything. Your CPU is capable of a lot - stop loading it with 
unnecessary garbage instructions because you dont like types.

I declare, programmers would benefit greatly by going back to the basics again. What follows
is the guide from entitlement towards enlightenment. 

# Data Packing
This section will go over the fundamental units to represent data, data types that emerge,
and common operations on that data.

## Fundamental Units
The fundamental units I will discuss is a bit, a byte, and a word.

### Bits 
Data is stored in 0's and 1's. A single 0 or 1 can only describe 2 states - 0 or 1. 
Looking at multiple 0's and 1's at once starts to paint a bigger picture. 
Let's just look at two bits for now.

```
00
```

If we consider this to be a number, there are 4 possible states, or combinations.

```
00
01
10
11
```

and that's it. We could assign a numerical value to each of these. Let's call the top 2-bit
combination 0, and follow the progression downwards for 1, 2, and 3. Notice that for the 
odd numbers (1 and 3) the rightmost bit is flipped, and even numbers (0 and 2) that bit 
is set to `0`. Additionally, the numbers that are less than 2 occur where the leftmost bit 
is set to `0`, and greater than or equal to 2 occur when the leftmost bit is flipped to `1`.
We can use this to form a basis for mapping binary values to numbers. The rightmost bit 
is assigned an index, starting from 0. Every bit to the left is given the next incremented 
index. Calculating the number that that bit represents is done by taking 2 to the power 
of that index. In that case we can see that the rightmost bit corresponds to 2^0, or 1, and 
the bit to its left corresponds to 2{{< super "1" >}}, or the number 2. If that bit is set to 1, then 
that number is present - if that bit is 0, then that number is not present. The value 
the binary number represents is 2^idx for every index that equals 1. Reread that last sentence.
Let's see what that looks like:

| binary | left value | right value | total value |
| - | - | - | - |
| 00 | 0 | 0 | 0 |
| 01 | 0 | 1 | 1 |
| 10 | 2 | 0 | 2 |
| 11 | 2 | 1 | 3 |

Expand that to 4 bits, and calculate the value for this number:
`1101`
Need help?
```
1101
= 2^3 + 2^2 + 0 + 2^0
= 8 + 4 + 0 + 1 
= 13
```
That's all there is to binary numbers. Expand all the way out to modern 64 bit integers, and 
your max value is `2^64 - 1` - exactly what you'd expect by applying the given formula.

### Bytes
A byte is a collection of 8 bits. So if we follow the number example, what do you 
think the max number that can be represented by a byte? Well, if every byte was filled 
in you'd have `2^7 + 2^6 ... + 2^1 + 2^0`. Which simplifies to `2^8 - 1`, which is 255.
See, they're simple.

This is, for the most part, the lowest size of data your computer cares about. In fact, 
memory addresses obey byte boundaries. CPUs have evolved to be very efficient to work 
with a whole byte rather than 1 bit at a time. In fact, a 64 bit CPU works with 64 
bits at a time, which is just 8 bytes.

### Word
A word is the number of bits that your CPU fits in a register. The word size is unique 
to the computer you are working on. Most modern CPUs use 64 bit registers, while 32 bit 
CPUs were quite common 10 years ago. Even if you are only operating on a single byte of 
data, your CPU will load a full 64 bit slice of data into its register. This is done 
for efficiency of doing many things quickly on a lot of data.

## Data Types 
The data types we will cover are signed and unsigned integers, floats and doubles, chars,
arrays, strings, and structs.

### Integers
First, we'll start with unsigned integers, hereby referred to as "uint". These are numbers 
that start at 0, and have a maximum value consistentn with the number of bytes. Common uints 
include uint8, uint32, and uint64, where the number refers to the total number of bits.
A uint16 is sometimes used, although uncommon. The only place I can immediately think of 
is specifying the number of network ports available on a computer. In that case, the ranges 
are as follows:

| Type | Min | Max |
| ---- | --- | --- |
| uint8 | 0 | 255 |
| uint16 | 0 | 65,535 |
| uint32 | 0 | 4,294,967,295 |
| uint64 | 0 | 18,446,744,073,709,551,615 |

Signed integers are only slightly different. Sometimes we may want to use negative integers;
this requires knowledge of an integer being positive or negative. So, a single bit is 
used to denote the sign - positive, or negative. This changes the ranges available:
| Type | Min | Max |
| ---- | --- | --- |
| int8 | -128 | 127 |
| int16 | -32768 | 32767 |
| int32 | –2,147,483,648 | 2,147,483,647 |
| int64 | -2^63 | 2^63 - 1 |

### Floating Point Numbers
Floats are named because they have a floating number of decimals. This may not make 
sense, but you probably already know something about floating point numbers. Scientific notation
for numbers exhibit this behavior. Think of a scientific number - what is it? I thought of
1.234 * 10^5. From now on, I'm going to use the e notation where "e" is a substitute for 
"* 10^". So our number can be rewritten as 1.234e5. This is how you can think of floats!
There have been multiple ways to represent a float over the years, but eventually the 
IEEE 754 standard came along to define a common float implementation (a history lesson
will illustrate how horrible things were before IEEE 754). 

Floats consist of 3 main parts - 1 sign bit, 8 exponent bits, and 23 bits fraction 
bits. Some super smart people came up with a way to implicitly store an extra bit 
in the fraction component, which doubles the number of values that can be mapped 
by a float.

I don't think it's exactly needed to understand how a float works, other than it 
is a number that behaves like scientific notation. The fraction part has limited
amount of precision. The number 1.23456789e10 and 1.23456789 would look very similar
in binary - just the exponent bits would change. If you get too precise, the float 
will not have enough precision to accurately store the number. For example, if you need 
centimeter accurate position when given a latitude, longitude and altitude coordinates for 
a point on earth, the earth size is too large relative to the size of a centimeter to 
accurately store in a float. That centimeter would probably get rounded to something 
closer to the nearest meter, or some fraction of a meter. When you need more precision,
a double is needed.

### Doubles
A Double is a floating point number with double precision. The IEEE 754 standard also 
establishes a standard for a double - it has 1 sign bit, 11 exponent bits, and 52 fraction
bits. As the fraction bits specify the precision of the number stored, that's a huge increase 
in the precision able to be stored in a double. There also exist Quadruples (128 bits) and 
Octuples (256 bits), but I've never used or had a need to use them.

### Chars
A char in C and C++ is actually quite simple. In C, you could do something like this:
```C
char a = 'a';
```
That would assign the value 'a' to a. But how is that data stored? Well, a char is 
a single byte, with values from -128 to 127. It's basically just an int8. In fact,
this is a great time to talk about character encodings. Encodings are just an established 
way to map one data representation to a binary data representation. The most basic character 
encoding is known as ASCII - [here's](https://www.asciitable.com/) an ASCII table for reference.

If we look at the ASCII table, we can see the lowercase alphabet numbers span from 97-122, 
uppercase from 65-90, and digits from 48-57. This is the simplest way to store text as binary 
data! The even greater expansion to ASCII is known as UTF-8, which is a super set of ASCII, but 
we'll save that for later. Just know that for now, when you type 'a' into your computer, it is
converted to the binary value 97.

### Arrays
An array is multiple data members of the same type in a sequence. For example:
```C
unsigned int time_series[10];
```
defines an array of 10 uints. If you were to pull up the space in memory where this 
data exists, you'd see the memory that this array takes up is `10 * sizeof(unsigned int)`
wide. Uints are 4 bytes, so 10 * 4 bytes would be 40 bytes total. Initializing an 
array can be done on definition, or by value.
```C
unsigned int time_series[10] = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
time_series[5] = 100; // assigns the element at index 5 to equal 100
```
The first initialization declares the variable, and sets each value. The second line 
looks up the 5th element (individual slot in an array) and sets it equal to 100.
Array indexes start at 0, so the index 5 would actually be the 6th slot. Arrays 
are an efficient way to store data and are a fundamental building block of many 
data structures.

### Strings
A string in C is a block of text, which is pretty much just arrays of chars. They are also 
null terminated - the end of the string is denoted by the null character, or the character 
that equals 0 in binary. This means that to find the length of a string, you have to loop 
through each index until you find one that equals 0 - the size is not known until you loop 
through the entire array. Strings in C are specified by using double quotes (") while single 
chars are denoted by ('). Therefore, the following is equivalent:
```C
char greeting1[] = {'H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd', '!', '\0'};
char greeting2[] = "Hello World!";
```
The first line explicitly adds the null termination character - '\0'. The second uses 
a feature of the C compiler - a null char is implicitly added to the end of string 
literals in double quotes. Strings in C++ is mostly just a wrapper around an array of
bytes.

### Structs
There are times where you will want to structure your data. Take this example:
```C
void print_color(int red, int green, int blue) {
    printf("red: %d, green: %d, blue: %d", red, green, blue); // %d prints an integer
    return;
}

int red = 10;
int green = 186;
int blue = 181;

print_color(red, green, blue);
```
We have a red value, a green value, and a blue value. A composite color is made of 
red, green, and blue. How do we make a composite color? We have three separate variables,
but no way to combine them. Defining a struct can allow us to make a data structure 
with known members (the variables that make up a struct).
```C
struct Color {
    int red;
    int green;
    int blue;
}; // creates a structure of type "struct Color"

void print_color(struct Color c) {
    printf("red: %d, green: %d, blue: %d", 
        c->red,    // accessing data from a pointer requires dereferencing
        c->green,  // the memory address. The operator for this is `->`
        c->blue);
    return;
}

struct Color color = { 10, 186, 181 }; // declares a variable named "color" with type "struct Color"

print_color(color);
```
Now rather than sending 3 arguments, we send just one. Defining a struct is defining a 
type. In C, the type we just declared is "struct Color", and now we have a better way 
to represent a color. To a computer, structs are just a block of memory that multiple 
variables are laid into, side by side. This is referred to as "a continguous block of 
memory". We'll get into the weeds about memory later.

In C++, you can attach functions to structs. This means that classes in C++ are basically 
just structs, and structs are basically just classes.

### Pointers 
Don't worry, I didn't forget about pointers. These deserve their own section, but we 
should learn more about memory before diving into pointers.

## Memory
There's 2 different types of memory that you'll hear about: stack, and heap. Usually 
stack memory is easily accessible, and pointers are stored on the stack, but often point 
to a heap memory location.

### What is Memory Anyways?
If you've heard about how much RAM (Random Access Memory) a computer has, that's what 
we're talking about. Memory is used to store data while running a program. Memory is 
indexed with a memory address. In 64 bit operating systems, memory addresses are indexed 
by a 64 bit number. Every address maps to a physical byte of memory in your RAM 
hardware. When a program starts, the kernel gives a place in memory for the program to 
use and operate. Like this:
```C
int a = 420;
int b = 69;
```
The assembly for this moves 420 to a memory location offset 4 bytes from the stack pointer,
then moves 69 to a memory location offset 8 bytes from the stack pointer. Later if we were 
to reference a or b, they would be loaded from a memory location into a CPU register. 
That's what memory looks like at a basic level!

### Stack Memory
Stack memory is often called fast. It's quick to assign a value to stack memory, and is 
quick to reference a variable in the stack memory. Stack memory is stored LIFO (Last In,
First Out). There are other restrictions to the stack though. For one, memory stored here must 
be known at compile time. Dynamically sized memory (memory size not known until runtime) 
must be stored elsewhere, aka the heap. However, the newly created heap memory address must 
be known - that's what a pointer is - and can be stored on the stack, since the size of 
a memory addresses is known. Stack memory is automatically allocated and deallocated 
when entering or exiting a new subprogram.

### Heap Memory
Heap memory is dynamic. Memory is explicitly allocated and deallocated on the heap.
The most common way to allocate heap memory in C is by using `malloc` (memory allocate):
```C
int numbers[] = malloc(100 * sizeof(int));
```
Sizeof is a compile-time resolved number, that would return the size in bytes that the 
int type occupies. So an int is 4 bytes, and therefore 400 bytes are allocated on the 
heap, which is enough room for 100 int elements in the array. Memory is uninitialized.
That means that whatever the bytes were set to in that memory region to begin with 
is the same. There's also calloc (clear allocate), which allocates and zeros the entire 
memory region.

### How to use memory
Allocating heap memory is generally regarded as "slow". It really depends on your 
particular scenario though. Generally, try and avoid allocating memory in a loop 
if you can help it. If there's a way to preprocess that memory allocation, do it.

### Pointers
Pointers are saved memory addresses. That's it.

When memory is allocated on the heap, it is formless. Its type is `void*`. You can give 
it form however - you can cast a `void*` pointer to a type. Let's take our previous 
struct example:
```C
struct Color {
    int red;
    int green;
    int blue;
};

void print_color(struct Color* c) {
    printf("red: %d, green: %d, blue: %d", c.red, c.green, c.blue);
    return;
}

struct Color color = { 10, 186, 181 }; // declares a variable named "color" with type "struct Color"

print_color(&color); // placing a `&` before a variable accesses the memory address
```
This is a far more common way the example would be implemented. The main advantage is 
that no data is copied. In the previous example, where the `struct Color c` was sent 
as a function argument, the struct would actually be copied into a local variable for 
the function to use. By sharing a pointer, the function can access the shared memory 
directly. Also, it could modify the function directly. Consider the following example:
```C
void modify_color(struct Color* c) {
    c->red = 255;
    c->green = 255;
    c->blue = 255;
}

struct Color color = { 0, 0, 0 };

modify_color(&color);
print_color(&color);
```
Since the pointer itsself is shared, the memory location that is modified is the same 
that is printed. Therefore, running the previous program would print
```bash
red: 255, green: 255, blue: 255
```
An important property about structs is that their structure is known at compile time.
This means that the offsets for struct members is known as well. Therefore, when 
accessing a struct members, the computer actually looks into the pointer + offset,
and loads that data. In `struct Color`, red is the first member, so the memory offset 
is 0. Remember that an int is 4 bytes, so for green the actual memory is `c + 4`, 
and blue would be `c + 8`. Oh yeah, by the way, pointer arithmetic is a thing. If 
I want to look at the memory 4 ahead of an address memory that I know, I can just 
add 4 to that memory address, and dereference the pointer to get the underlying data.

Functions and their instructions hold a place in memory. Recall that functions can 
take pointers as arguments. Therefore, you can pass a pointer to a function as an 
argument. Check this out:
```C
#include <stdio.h>

void print_num(int n) {
    printf("%d", n);
}

void do_something_with_num(void (*f)(int)) {
    int number = 100;
    f(number);
}
int main(void) {
    do_something_with_num(print_num);
}
```
Running this code would print `100` to the console.

### Memory Footguns
The 2 most common memory footguns are memory leaks, and segmentation faults.

#### Memory Leaks
Dynamic memory is allocated on the heap. It must be explicitly deallocated. Imagine this 
code:
```C
while (1) {
    void* explodingMemory = malloc(1);
}
```
This code allocated 1 byte of memory at a time, forever. It will take over your 
entire system resources. Alternatively, consider this code:
```C
for (int i = 0; i < 100; i++) {
    void* explodingMemory = malloc(1);
}
```
This code does exit the loop. But, the memory is still not being deallocated. This 
is a memory leak, and can eventually hog system resources of memory continues to leak 
over time. Memory leaks are the cause of around 70% of bugs in production software.

#### Segmentation Fault
A segmentation fault occurs when you try and do something to memory that the operating 
system doesn't like. This is usually caused by reading or writing to some memory 
location the OS did not give the program access to. There's an obvious security vulnerability
being prevented here - if malware was on the system, and could see data at any memory location,
it could spy on programs and log that information.

If you index too deep into an array (further than the memory allocated would allow),
you get a segmentation fault but ONLY if the memory after that location is not owned 
by you. Segmentation faults do not "throw" an error, they completely crash your program. 
Handle them, and check that your pointers are not zero before reading and writing to
them.

### Slow memory operations
Allocating takes time. Copying takes time. Copying is often convenient, but not 
always necessary.

## Essential Data Structures
[Click here](https://frontendmasters.com/courses/algorithms/) for a fantastic free 
course on data structures and algorithms.
