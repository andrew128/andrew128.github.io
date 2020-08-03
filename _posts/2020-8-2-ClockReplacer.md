---
layout: post
title: Clock Replacer Algorithm in C++
---
This post introduces the Clock Replacer algorithm and an implementation in C++.

## Post Outline
- [Motivation](#motivation)
- [Explanation](#conceptual-explanation)
- [Code](#code)
- [Resources](#resources)

## Motivation
The Clock Replacer algorithm is a page replacement algorithm.
Page replacement is the database problem of deciding what pages to hold in memory as opposed to on disk.

The purpose of a clock replacer is to allow the client, restricted by storage size, to only store the most important items by removing pages that will not be needed in the near future.
It accomplishes the same purpose as the more well-known LRU (least recently used) cache.

## Explanation

The general idea of the algorithm is that a client can add a new page at any point in time. 
If there is room in the clock replacer, it will simply be added.
Otherwise the new item replaces the least used item.
Least used is defined by the clock replacer algorithm.

### Basic Structure and Algorithm
Let's picture the clock replacer as a circular buffer.
Each slot in the circular buffer represents a frame, which can hold a single page at any point in time.
Each slot also has a reference bit.
When a page is first added or accessed afterwards, the reference bit is set to 1.

There is a clock pointer that rotates around the buffer, which decides the next item that should be evicted.
When an item must be evicted, the clock pointer checks the frame it is currently pointing at, setting the reference bit from 1 to 0.
It then increments to the next frame in a circular manner.
If the reference bit is already 0, the clock pointer identifies the frame for eviction.

### Pinning and Unpinning
In addition to adding a page, the client can also pin pages to ensure they will never be victimized.

## Code

The code is in [this repo](https://github.com/andrew128/ClockReplacer).
To run the tests, use the following command:
```
g++ -std=c++17 -g -Wall -W -o a.out clock_replacer_test.cpp clock_replacer.cpp && ./a.out
```

## Resources
- [CMU Databases 2019](https://15445.courses.cs.cmu.edu/fall2019/project1/)

