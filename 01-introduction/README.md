# Introduction

## Why Is Software Slow

Modern software is massively inefficient. Here's a table to show a program that multiplies two
4096-by-4096 matrices.

```
+---------+-----------------------------+------------------+-----------------+
| Version | Implementation              | Absolute speedup | Relative speedup|
+---------+-----------------------------+------------------+-----------------+
| 1       | Python                      | 1                | —               |
| 2       | Java                        | 11               | 10.8            |
| 3       | C                           | 47               | 4.4             |
| 4       | Parallel loops              | 366              | 7.8             |
| 5       | Parallel divide and conquer | 6,727            | 18.4            |
| 6       | plus vectorization          | 23,224           | 3.5             |
| 7       | plus AVX intrinsics         | 62,806           | 2.7             |
+---------+-----------------------------+------------------+-----------------+
```

### What prevents system from achieving optimal performance by default?

#### CPU Limitation

Increasing CPU can be good, but can't solve slow code / redundant code.

Ex: If we wrote Bubble Sort, CPU can't attempt to run a better sorting.

#### Compiler Limitation

Compilers are great at
eliminating redundant work, but when it comes to making more complex decisions
like vectorization, they may not generate the best possible code.

Additionally, compilers
cannot perform optimizations unless they are absolutely certain it is safe to do
so.

#### Algorithmic complexity analysis limitations

Even though Insertion sort is O(N^2) and Quick Sort is (N log N). On small size data such as 50 elements, Insertion Sort wins. So please don't blindly follow complexity without benchmarking.

#

Coding practices that prioritize code clarity, readability, and
maintainability can reduce performance.

Highly generalized and reusable code can
introduce unnecessary copies, runtime checks, function calls, memory allocations, etc

For instance, polymorphism in object-oriented programming is usually implemented
using virtual functions, which introduce a performance overhead

## Why care about performance?

During PC era, the one who paid the slow code is end user.

Now, when Cloud era comes, software provider need to pay that, slow code means more electricity to consume, means more server to serve, etc.

Google reported that a 500-millisecond delay in search caused a 20%
reduction in traffic.

For Yahoo! 400 milliseconds faster page load caused 5-9% more
traffic.

Slower the service works, less people will use it.

## What Is Performance Analysis?

Performance analysis is a process of collecting information about how a program
executes and interpreting it to find optimization opportunities

## What Is Performance Tuning?

Locating a performance bottleneck is only half of an engineer’s job. The second half
is to fix it properly.

To take advantage of all the computing power of modern CPUs, you need to understand
how they work

This is a type of optimization that
takes into account the details of the underlying hardware capabilities (Low Level optimization).

To successfully implement
low-level optimizations, you need to have a good understanding of the underlying
hardware.