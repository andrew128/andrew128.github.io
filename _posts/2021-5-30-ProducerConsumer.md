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

Note that the thread calling `notify()` doesn't need to acquire the mutex of the waiting thread due to the `notify()` function itself.

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

The `produce()` function is where we implement the addition of data into an instance of the `Buffer` class in a thread safe manner.
At a high level, we acquire a unique lock on the mutex, which is required by condition variables (as opposed to a basic std::mutex b/c of the condition variable function signatures).

We then use the function `wait(lock, pred)` on the `not_full` condition variable.
We want to wait if the buffer is full and only continue if another thread notifies us that the buffer is not full anymore.
The above function signature is equivalent to (as specified in the [docs](https://en.cppreference.com/w/cpp/thread/condition_variable/wait)):

```
while (!pred()) {
    wait(lock);
}
```

Essentially the wait will only occur if the predicate is false.
Since we want to only wait if the buffer is full, our predicate will return true if the buffer is not full.

The code after the wait will only fire if the buffer is not full.
We next need to add the input to the buffer and update the appropriate fields of the `Buffer` instance, mainly the size and the right index.

At this point, we have finished modifying the `buffer` field and no longer need the lock, so we unlock it.

We then call `notify()` with the `not_empty` condition variable because the buffer can't be empty as we have just added a value.
This will notify any waiting consumer threads that may be waiting if the `buffer` was empty.

Below is the full `produce()` function code:

```
void Buffer::produce(int thread_id, int num) {
    // Acquire a unique lock on the mutex
    std::unique_lock<std::mutex> unique_lock(mtx);
    
    std::cout << "thread " << thread_id << " produced " << num << "\n";
    
    // Wait if the buffer is full
    not_full.wait(unique_lock, [this]() {
        return buffer_size != BUFFER_CAPACITY;
    });
    
    // Add input to buffer
    buffer[right] = num;
    
    // Update appropriate fields
    right = (right + 1) % BUFFER_CAPACITY;
    buffer_size++;
    
    // Unlock unique lock
    unique_lock.unlock();
    
    // Notify a single thread that buffer isn't empty
    not_empty.notify_one();
}
```

### Buffer::consume()

We now cover the thread safe `consumer()` function of the buffer.
It is essentially the exact opposite of the `Buffer::produce()` code.
We acquire the lock and only wait if the buffer is empty.
We then remove the next value to remove from the buffer and update the appropriate fields.

We end by unlocking the lock, notifying any thread that might be waiting that the buffer isn't full anymore, and returning the value we consumed.

```
int Buffer::consume(int thread_id) {
    // Acquire a unique lock on the mutex
    std::unique_lock<std::mutex> unique_lock(mtx);
    
    // Wait if buffer is empty
    not_empty.wait(unique_lock, [this]() {
        return buffer_size != 0;
    });
    
    // Get value from position to remove in buffer
    int result = buffer[left];
    
    std::cout << "thread " << thread_id << " consumed " << result << "\n";
    
    // Update appropriate fields
    left = (left + 1) % BUFFER_CAPACITY;
    buffer_size--;
    
    // Unlock unique lock
    unique_lock.unlock();
    
    // Notify a single thread that the buffer isn't full
    not_full.notify_one();
    
    // Return result
    return result;
}
```

## Producer Consumer Code
We now write the code to actually call the `Buffer` class's methods.
To start, we can create an instance of the `Buffer` class.
The `srand` line is simply to randomize the seed for the produce function, which adds random numbers to the buffer.

We then create several produce and consume threads that call `produceInt` and `consumeInt` (described below).
We finally call join on all the threads to wait for completion.

```
int main(int argc, const char * argv[]) {
    std::cout << "Executing code in main...\n";
    
    // Initialize random seed
    srand (time(NULL));
    
    // Create Buffer
    Buffer buffer;
    
    // Create a thread to produce
    std::thread produceThread0(produceInt, std::ref(buffer));
    
    std::thread consumeThread0(consumeInt, std::ref(buffer));
    
    std::thread produceThread1(produceInt, std::ref(buffer));
    
    std::thread consumeThread1(consumeInt, std::ref(buffer));
    
    std::thread produceThread2(produceInt, std::ref(buffer));
    
    produceThread0.join();
    produceThread1.join();
    produceThread2.join();
    consumeThread0.join();
    consumeThread1.join();
    
    std::cout << "Done!\n";
    return 0;
}
```

The `produceInt()` function takes as input a reference to the `Buffer` class instance and generates a random number between 1 and 10 to add to the buffer, calling the buffer's `produce()` method.

```
// Takes in reference to a buffer and adds a random integer
void produceInt(Buffer &buffer) {
    for (int i = 0; i < 4; i++) {
        // Generate random number between 1 and 10
        int new_int = rand() % 10 + 1;
        buffer.produce(i, new_int);
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}
```

The `consumeInt()` function acts similarly except that it consumes a number from the buffer instead using the buffer's `consume()` method.
```
// Takes in reference to a buffer and returns the latest int added
// in the buffer
void consumeInt(Buffer &buffer) {
    for (int i = 0; i < 6; i++) {
        buffer.consume(i);
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}
```

Note that the number of values produced equals the number of values consumed (otherwise the code would not complete because either a produce thread or a consume thread would be waiting for an event that would never happen).

Here is a sample output:
```
Executing code in main...
thread 0 produced 7
thread 0 consumed 7
thread 0 produced 9
thread 0 consumed 9
thread 0 produced 7
thread 1 produced 5
thread 1 consumed 7
thread 1 produced 4
thread 1 consumed 5
thread 1 produced 8
thread 2 produced 10
thread 2 consumed 4
thread 2 produced 1
thread 2 consumed 8
thread 2 produced 5
thread 3 consumed 10
thread 3 consumed 1
thread 3 produced 10
thread 3 produced 7
thread 3 produced 9
thread 4 consumed 5
thread 4 consumed 10
thread 5 consumed 7
thread 5 consumed 9
Done!
Program ended with exit code: 0
```

## Resources
- [Code used in this blog post](https://github.com/andrew128/ProducerConsumer)
- [Baptiste Wicht's Blog post](https://baptiste-wicht.com/posts/2012/04/c11-concurrency-tutorial-advanced-locking-and-condition-variables.html)
- [Back to Basics Concurrency Talk Slides by Arthur O'Dwyer](https://github.com/CppCon/CppCon2020/blob/main/Presentations/back_to_basics_concurrency/back_to_basics_concurrency__arthur_odwyer__cppcon_2020.pdf)
- [C++ Documentation on Condition Variables](https://www.cplusplus.com/reference/condition_variable/condition_variable/)
- [Blog Post by Domi Yan on the Producer Consumer Problem](https://levelup.gitconnected.com/producer-consumer-problem-using-condition-variable-in-c-6c4d96efcbbc)