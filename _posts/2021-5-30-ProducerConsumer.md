---
layout: post
title: The Producer Consumer Problem in C++
---

We will go over a solution to the Producer Consumer problem in concurrency with multiple producers and consumers in a buffer of bounded size.
The solution is written in C++ and uses mutexes and condition variables.
This post is based off of the blog post [here](https://baptiste-wicht.com/posts/2012/04/c11-concurrency-tutorial-advanced-locking-and-condition-variables.html) by Baptiste Wicht.

## Post Outline
- [Background](#background)
- [Bounded Buffer](#bounded-buffer)
- [Producer Consumer Code](#producer-consumer-code)
- [Resources](#resources)

## Background

### Producer Consumer Problem Setup

In the Producer Consumer problem, many producers are adding data to a data structure (i.e. buffer) that many consumers are reading from at the same time (i.e. concurrently).
The heart of the problem lies in coordinating the producers to only add data if there is space in the buffer and the consumers to only remove data from the buffer if there exists data in the buffer. 

An initial solution to the data race problem may use while loops to check the size. 
For example, the producer may loop while the buffer is full and when there is space, exit the while loop and add data to the buffer.
However, this solution will result in data races.

Data races easily occur if no synchronization techniques are used because the size of the buffer needs to be updated atomically with the production and consumption of data in the buffer.
If the buffer size update is not thread safe, producers may try to add data to the buffer when it is already full and consumers may try to remove data from the buffer when the buffer is empty.

We will go over two synchronization primitives, **mutexes** and **condition variables**, that can be used in one possible solution to this problem.

### Mutexes
Mutexes are a synchronization primitive that prevent data races when multiple threads access the same data structure.
We can make transactions thread safe using mutexes by locking a particular block of code that we want to execute such that only one thread is accessing that block at a time.
This is done by calling `mutex.lock()` at the start of the block and `mutex.unlock()` at the end.
Typically mutexes are implemented with a queue of threads.
Threads wanting to access a thread safe block of code call `mutex.lock()` and if another thread is currently accessing that block, the thread is added to a queue from which threads are removed one at a time and run when the running thread calls `mutex.unlock()`.

Mutexes alone are enough to solve the data race problem discussed previously if we use the while loops to check for the buffer being empty or full and use mutexes to make the code thread safe.
However, the solution is inefficient because of the while loop, which keeps the CPU busy unnecessarily while it loops.
We can call a sleep function in intervals but ultimately this solution is still inefficient because the threads can't talk to each other.

Note that with mutexes, **all** accesses to a shared data structure between threads must be synchronized with mutexes.
Otherwise, data races will occur.

### Condition Variables
As a solution to the previous problem, we will introduce condition variables, which allow inter thread communication.

Condition variables work by making a thread wait (i.e. blocked) until it is notified to resume by another thread.
A unique lock is used to lock the calling thread that calls `wait()`.
The calling thread remains blocked until another thread calls `notify()` on the same condition variable.

When a calling thread calls `wait()` it must first acquire the mutex as input to `wait()`.
During `wait()`, the thread is placed on the condition variable's queue of threads and the mutex is released.
Once the thread that is waiting is notified by another thread, it reacquires the mutex.

## Bounded Buffer

Let's go over the header file for the `Buffer` class.
The main part of the problem is the `Buffer` data structure, which has two functions for "producing" (i.e. placing integers in the buffer) and "consuming" (i.e. removing integers from the buffer).

The buffer is a circular bounded buffer, meaning that it has a fixed size as defined by the macro `BUFFER_CAPACITY`.
The buffer also rotates in a circular fashion as kept track of by the left and right integer index fields of the `Buffer` class.

We also have a mutex for the condition variables `not_empty` and `not_full`.
Note that we have a single mutex (instead of one for each condition variable) because we want to synchronize the `produce()` and `consume()` functions.
We will go into more detail about the usage of the condition variables in the following sections.

```
#define BUFFER_CAPACITY 10

class Buffer {
    // Buffer fields
    int buffer [BUFFER_CAPACITY];
    int buffer_size;
    int left; // index where variables are put inside of buffer (produced)
    int right; // index where variables are removed from buffer (consumed)
    
    // Fields for concurrency
    std::mutex mtx;
    std::condition_variable not_empty;
    std::condition_variable not_full;
    
public:
    // Place integer inside of buffer
    void produce(int thread_id, int num);
    
    // Remove integer from buffer
    int consume(int thread_id);
    
    Buffer();
};
```

### Buffer::produce()

### Buffer::consume()

## Producer Consumer Code

// go over main() code

## Resources
- [Baptiste Wicht's Blog post](https://baptiste-wicht.com/posts/2012/04/c11-concurrency-tutorial-advanced-locking-and-condition-variables.html)
- [Back to Basics Concurrency Talk Slides by Arthur O'Dwyer](https://github.com/CppCon/CppCon2020/blob/main/Presentations/back_to_basics_concurrency/back_to_basics_concurrency__arthur_odwyer__cppcon_2020.pdf)
- [C++ Documentation on Condition Variables](https://www.cplusplus.com/reference/condition_variable/condition_variable/)
- [Blog Post by Domi Yan on the Producer Consumer Problem](https://levelup.gitconnected.com/producer-consumer-problem-using-condition-variable-in-c-6c4d96efcbbc)