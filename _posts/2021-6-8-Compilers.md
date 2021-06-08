---
layout: post
title: An Introduction to Compilers
---
This blog post introduces the world of compilers with an example in Golang.
The code comes from the book [Writing An Compiler In Go](https://compilerbook.com/) by Thorsten Ball.

## Post Outline
- [Introduction](#introduction-and-background)
- [Background]
- [Concrete Code]()
- [Resources]

## Introduction
In this blog post we will be covering the main parts of writing a compiler.
The post is divided into two main parts: background (where I go over the basic outline and details of compilers) and code (where I bring up specific examples from the book Writing a Compiler in Go).

## Background

### How do compilers work?

In general, compilers take in source code and output machine code that can be executed on a specific architecture.
More specifically, compilers take in source code and parse it into an AST (abstract syntax tree as covered in a [previous blog post](https://andrew128.github.io/Interpreters/)) using a lexer and a parser.
The AST is then converted into an intermediate representation upon which several optimization passes are applied to.
Then a code generator outputs the target machine code.
There are many more steps involved but these ones cover the major parts.

INCLUDE PICTURE OF GENERAL PROCESS

### How do compilers compare to interpreters?

Interpreters, like compilers, are another common kind of language processor.
Both will have the same front end where the source code is lexed and parsed into an abstract syntax tree (AST).
After the AST is where the paths diverge.
An interpreter directly executes the operations specified in the source program on user supplied inputs
A compiler, on the other hand, produces a target program in the target language that is executed when the user supplied inputs.

A compiler-produced program is typically much faster than an interpreter because the machine code from the compiler program is optimized for the specific architecture. 
On the other hand, an interpreter can be much better at providing error messages that are language specific to the source code because there isn't the source/target language abstraction layer that the compiler has.

### What about languages that are both compiled and interpreted?

There are many popular languages like Java and Python that are both compiled and interpreted.

The source program in these languages is first compiled into an intermediate program represented in bytecode.
The bytecode is then interpreted using a processor virtual machine.

Because of the two separate components to the overall process, the bytecodes can be compiled on one machine and interpreted on another, introducing some flexibility. 

### Virtual machines

Let's go into more detail about the processor virtual machine.

Note that the processor virtual machine (like JVM) is different from a system virtual machine (like a VMware product).
A system virtual machine is a program that emulates a computer, including the disk drive, hard drive, graphics card, and more.
A processor virtual machine is used to implement programming languages and are used to emulate the behavior of the hardware.
They can be thought of as the software equivalent of the hardware.
More detail can be found in [this wikipedia article](https://en.wikipedia.org/wiki/Virtual_machine).

#### Hardware: the von Neumann architecture
What exactly is the hardware that a processor virtual machine is mimicking?
The hardware that is being mimicked, while varying depending on the brand and model, typically follows the von Neumann architecture.

The von Neumann architecture has the following parts:
- CPU
    - processing unit: ALU + Processor Registers
    - control unit: instruction register, program counter
- Memory (RAM)
- Mass storage (hard drive)
- input/output devices

The CPU (i.e. the central processing unit) is made up of two parts: the processing unit and the control unit.
The processing unit has the ALU (arithmetic logic unit) and multiple processor registers.
The control unit has a instruction register (holds current instruction being executed) and a program counter (keeps track of next instruction to execute).
The CPU performs "basic arithmetic, logic, controlling, and input/output operations specified by the instructions in the program" ([Source](https://en.wikipedia.org/wiki/Central_processing_unit)).
The memory (Random Access Memory) stores the program's data and instructions.
There is also mass storage in the form of hard drives and disks that make up non volatile memory.
Then there are input/output devices like the keyboard and the display.

// INCLUDE PICTURE FROM WIKIPEDIA OF VON NEUMANN ARCHITECTURE (https://en.wikipedia.org/wiki/Von_Neumann_architecture)

#### CPU

The CPU performs a constant fetch, decode, execute cycle.

When a CPU is measured in megahertz, it refers to the frequency of this cycle (i.e. how many cycles in a particular time frame).
During the **fetch** phase, an instruction is fetched from memory using the program counter.
During the  **decode** phase, an instruction is decoded, meaning that the computer determines which operation should be executed based off of the instruction.
In the **execute** phase, the instruction is executed. 
This can mean that the values stored inside of registers is modified, that data is transferred from a register to memory, that data is moved around in memory, that output is generated, that input is read, and more.

#### Memory

So the CPU uses a program counter to keep track of where to fetch the next instruction.
The program counter's value is an address that points to somewhere in memory.

Memory is organized into groups of bytes called words.
The size of the word varies depending on the architecture.
Typically 32 and 64 bit word sizes are used in modern computers.


// call stack

// processor registers

#### Physical Machine to Virtual Machine

Now that we know what the virtual machine is emulating, lets see how the virtual machine mimics the hardware. 
The CPU performs

#### Stack based VM vs Register based VM

### The compiler in the book
The compiler is for the book's `Monkey` language and is written in Golang.
The compiler we're writing uses a stack based virtual machine.

## Code

### Bytecode

// endian

// 
### Compiling Expressions

### Conditionals

## Resources
- Writing a Compiler in Go
- Compilers: Principles, Techniques, and Tools
- The Elements of Computing Systems