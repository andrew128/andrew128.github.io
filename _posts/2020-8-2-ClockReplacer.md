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
Besides adding and victimizing frames, we can also pin and unpin frames.
Pinning a frame means that the page id in that frame will not be considered for eviction until it is unpinned.

We will represent the replacer using 3 boolean vectors, *inClockReplacer*, *clockRefFlag*, and *pinned*.
If *inClockReplacer* is true at a particular index, that means that index (aka frame id) is holding a page. The vector *clockRefFlag* holds the reference bit and *pinned* stores whether the frame is pinned or not.

### Accessing A Frame
Accessing a frame means either adding a new page to that frame or reading from an existing page at the frame.
Either way, the code is the same.
We mark the frame as storing a page and set its reference flag to 1.

```
bool ClockReplacer::AccessFrame(int frame_id) {
  if (frame_id < 0 || frame_id >= numPages || pinned[frame_id]) return false;

  inClockReplacer[frame_id] = true;
  clockRefFlag[frame_id] = true;
  numFramesInClockReplacer++;
  return true;
}
```

### Victimizing A Frame
Let's picture the clock replacer as a circular buffer.
Each slot in the circular buffer represents a frame, which can hold a single page at any point in time.
Each slot also has a reference bit.
When a page is first added or accessed afterwards, the reference bit is set to 1.
A clock pointer (an integer) loops around the buffer and sets the reference bit to 0. 
If the reference bit is already 0, then the item is chosen to be victimized.
We use the input pointer **frame_id* to hold the value of the frame id that was victimized.

```
bool ClockReplacer::VictimizeFrame(int *frame_id) {
  // Loop while there are still frames to be victimized
  while (numFramesInClockReplacer != 0) {
    // Only consider victimizing frame if page is stored there and 
    // frame is not pinned
    if (inClockReplacer[clockPointer] && !pinned[clockPointer]) {
      if (clockRefFlag[clockPointer]) {
        // Set reference bit to 0
        clockRefFlag[clockPointer] = false;
      } else {
        // Identified frame to be victimized
        *frame_id = clockPointer;
        inClockReplacer[clockPointer] = false;
        numFramesInClockReplacer--;
        return true;
      }
    }
    clockPointer = (clockPointer + 1) % numPages;
  }

  return false;
}
```

### Pinning and Unpinning
We will use a boolean vector to keep track of which pages are pinned.
Pinning and unpinning are defined by the *Pin()* and *Unpin()* methods, which update the boolean vector *pinned*.

## Code

The full code is in [this repo](https://github.com/andrew128/ClockReplacer).
To run the tests, use the following command:
```
g++ -std=c++17 -g -Wall -W -o a.out clock_replacer_test.cpp clock_replacer.cpp && ./a.out
```

## Resources
- [CMU Databases 2019](https://15445.courses.cs.cmu.edu/fall2019/project1/)

