--- 
layout: post
title: "C++: C-Style arrays vs. std::array vs. std::vector"
author: "Seth Hendrick"
comments: true
category: "Coding Tips"
tags: [array, c++, c-style array, pointers, sizeof, stack, std-array, std-vector, vector]
---

Arrays are a very basic data structure used in programming.  Unlike linked lists, arrays are guaranteed have its elements contiguous in memory.  That is, someArray[0] is directly next to someArray[1] in memory.  However, in C++, there are three ways to use arrays:  C-style arrays, std::array (as of C++11) and std::vector.  What’s the difference between the three? Read on!

## C-Style Array:

When many people think of C++ arrays, they think of something to the effect of:

```c
int myArray[3] = {1, 2, 3};
```

This is a feature left over from C, so hence the name “C-Style array”.  But, C-style arrays are very primitive.  For example there is no easy way to determine the size of this array.  Sure, you can do:

```c
int myArray[5] = {1, 2, 3, 4, 5};
size_t arraySize = sizeof(myArray) / sizeof(int);
std::cout << "C-style array size: " << arraySize << std::endl;
// Outputs:
// C-style array size: 5
```

But this only works when the array is in the same scope as the size operation.  An array is just a pointer to the first element in the array.  That is dereferencing the array with the ‘*’ and doing myArray[0] return the same value (see example below).  Consequently, doing myArray[1] and *(myArray + 1) will also return the same thing.  That’s because under the hood, myArray[x] becomes *(myArray + x).  We use operator [] since its cleaner.

```c
std::cout << std::boolalpha << (myArray[0] == *myArray) << std::endl;
std::cout << std::boolalpha << (myArray[1] == *(myArray + 1) << std::endl;

// Outputs:
// true
// true
```

So when you pass an array into a function, the function just sees a pointer. If we try the same sizeof() thing we did above, we will get a very different outcome:

```c
#include <iostream>

void printSize(int someArray[5]) {
    std::cout << sizeof(someArray)/sizeof(int) << std::endl;
}

int main() {
    int myArray[5] = {1, 2, 3, 4, 5};
    printSize(myArray);
}

// Outputs:
// 2
```

Wait? WHAT! Where does 2 come from?  Like I said, an array is just a pointer.  To put it simply, when we pass an array into a function, it “loses” its array “properties” and becomes a pointer.  int someArray[5] in the parameter list becomes int *someArray.  A pointer is of size size_t, which on my computer is 8 bytes.  An int, meanwhile, is 4 bytes.  We are doing sizeof(size_t) / sizeof(int), which is 2.

So the only way to get the size of an array is to pass around a size_t with the array, which is annoying.

There is another limiting factor with C-style arrays.  We can not use C++11’s fancy foreach loop with them outside the same scope.  The below example works just fine:

```c
int main() {
   int myArray[5] = {1, 2, 3, 4, 5};
   for (int &i : myArray) {
       std::cout << i << ", " << std::endl;
   }
}
```

However, once we pass it into a function, such as below:

```c
#include <iostream>

void printElements(int someArray[5]) {
    for (int &i : someArray) {
        std::cout << i << ", " << std::endl;
    }
}

int main() {
    int myArray[5] = {1, 2, 3, 4, 5};
    printElements(myArray);
}
```

We get a very nasty compile error.

The solution to both of these problems is C++11’s new std::array.

## std::array

std::array is a very thin wrapper around C-style arrays that go on the stack (to put it simply, they do not use operator new.  The examples above do this).  Like arrays that go on the stack, its size must be known at compile time.

Constructing an std::array is easy.  Instead of C’s int someArray[x], its std::array<int, x> someArray, where x is the size of the array.  See the example below.  Note that when we pass in std::array into the function, we can iterate over it, and it knows its size!

```c
#include <array>
#include <iostream>

void printElements(const std::array<int, 5>; &someArray) {
    for (const int &i : someArray) {
        std::cout << i << ", " << std::endl;
    }

    std::cout << "Size: " << someArray.size() << std::endl;
}

int main() {
    std::array<int, 5> myArray = {1, 2, 3, 4, 5};
    printElements(myArray);
}

// Outputs:
// 1,
// 2,
// 3,
// 4,
// 5,
// Size: 5
```

Hurray! It worked!

## std::vector

Now, I hear some of you saying “But wait, what about std::vector? Doesn’t this do the same thing?”.  While it is true, you can swap out the above code with std::vector and get the same result, under the hood it does things differently.

std::array is a static array whose size is known at compile time.  It is a thin wrapper of c-style arrays that go on the stack.  std::vector is an entirely different beast.  It is a dynamic array that goes on the heap. Its size does not need to be known at compile time.  As you add or remove things from std::vector, the underlying array changes size.  std::array can not change size once it is created, just like c-style arrays that go on the stack.  std::vector can change its size once created, just like c-style arrays that go on the heap.

Think of std::array as a wrapper to:

```c
int someArray[5];
```

while std::vector is a wrapper to:

```c
int *someArray = new int[5];
```

You should use std::array when the array size is known at compile time.  You should use std::vector when you do not, or the array can grow.
