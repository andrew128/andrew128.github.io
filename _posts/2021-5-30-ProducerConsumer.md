---
layout: post
title: The Producer Consumer Problem in C++
---

## Intro

We will go over the C++ code solution to the classic Producer Consumer problem in concurrency.
The solution uses mutexes and condition variables.
This post is based off of the blog post [here](https://baptiste-wicht.com/posts/2012/04/c11-concurrency-tutorial-advanced-locking-and-condition-variables.html) by Baptiste Wicht.

## Mutexes
lock guard vs unique lock

## Condition Variables

## Bounded Buffer
The main part of the problem is the `Buffer` data structure, which has two functions for "producing" (i.e. placing integers in the buffer) and "consuming" (i.e. removing integers from the buffer).

it's a circular buffer

### Buffer::produce()

### Buffer::consume()

## Producer Consumer Code

## Resources
- https://baptiste-wicht.com/posts/2012/04/c11-concurrency-tutorial-advanced-locking-and-condition-variables.html
- https://github.com/CppCon/CppCon2020/blob/main/Presentations/back_to_basics_concurrency/back_to_basics_concurrency__arthur_odwyer__cppcon_2020.pdf
- https://levelup.gitconnected.com/producer-consumer-problem-using-condition-variable-in-c-6c4d96efcbbc
- https://www.cplusplus.com/reference/condition_variable/condition_variable/