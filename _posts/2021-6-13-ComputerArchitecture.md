---
layout: post
title: Computer Hardware and Architecture - A Bird’s Eye View
---

In this blog post we take a step lower from our previous blog posts on compilers/virtual machines and operating systems. It will string together how physics and hardware are built and used to execute our abstract thoughts and ideas illustrated in higher level languages.

## Post Outline
- [Introduction](#introduction)
    - [Operating Systems]()
    - [Compilers]()
- [Conclusion](#conclusion)
- [Resources](#resources)

## Introduction
Let's start with a brief overview of operating systems and compilers before working our way downwards.
Here are separate blog posts on the [compiler](https://andrew128.github.io/Compilers/) and the [operating system](https://andrew128.github.io/OperatingSystems/) that go into more detail.

### Operating Systems
At the top is the operating system, which executes user (application developer) programs.
The OS is a piece of software built on top of the hardware that abstracts away certain components from the user including process and thread management, the file system, and dealing with memory (instead presenting virtual memory to the user and user processes).

BIOS (Basic Input/Output System) is a set of computer instructions stored in read only memory that performs hardware initialization when the computer is first turned on and sets up the operating system for the user. 

### Compilers
Next is the compiler, which turns high level source code into low level machine code that an be executed on the hardware.
Operating systems are nowadays typically written in mid to high level source code (i.e. C/C++) although the first operating systems were written in machine code.
There are some languages like C/C++ that have a compiler that does this translation directly.
While this does have certain speed benefits, it is also less portable because a new compiler must be used for each different operating system.

Then there are languages like Java and its famed JVM.
Java compilation has two parts: the compilation of the source code into Java bytecode by the compiler and the execution of the bytecode that is interpreted by the JVM.
The Java compiler (i.e. the first step) is platform independent and the bytecode of a program can be used on any platform.
The JVM is platform dependent since it has to execute for a particular platform.
This breaks up the compilation process and allows for more modularity and flexibility.

## Assembler

### What is an assembler

### How does an assembler turn assembly into machine code

## CPU

### What does a CPU do

### What are the main components

## Machine Code

## Storing Data with Sequential Chips

### How is data stored

## ALU Computations

### How does an ALU perform computatoins

## Transistors

### How do transistors work

## Conclusion
We conclude our post at the transistor.

## Resources
- [The Elements of Computing Systems](https://www.amazon.com/Elements-Computing-Systems-Building-Principles/dp/0262640686)
- [Computer Systems: A Programmer's Perspective](https://www.amazon.com/Computer-Systems-Programmers-Perspective-3rd/dp/013409266X)