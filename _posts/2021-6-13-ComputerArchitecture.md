---
layout: post
title: Computer Hardware and Architecture - A Bird’s Eye View
---

In this blog post we take a step lower from our previous blog posts on compilers/virtual machines and operating systems. We will string together how physics and hardware are built and used to execute our abstract thoughts and ideas illustrated in higher level languages.

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

We get assembly (symbolic) language from the output of the compiler.
Assemblers translate programs written in this symbolic assembly language into binary machine language.
Symbolic assembly language examples include `PUSH 0x3` or `MOV 0x0 %eax` (examples from x86).
For more examples of assembly (specifically x86) and how it is used in when compiling C source code, check out [this blog post](https://andrew128.github.io/x86-binary/).
Let's start by going over what machine code is.

### Binary Machine code
Machine code is a series of 1s and 0s that form instructions (and their operands).
These instructions make changes to memory using the CPU (i.e. its ALU and the registers).
In a 32 bit computer, instructions are 32 bits long.
To understand a specific computer's machine language, we need to know its ISA (instruction set architecture).
The ISA is the blueprint of a processor.
There are three general categories of commands that an ISA should specify:
- Arithmetic and logic operations (e.g. addition, XOR, etc.)
- Memory access
- Control flow (e.g. looping, finding next instruction, conditionals like if statements)

Let's take a look at the specific example architecture for the Hack Machine Language introduced in the book `The Elements of Computing Systems`.
The architecture is a von Neumann architecture, meaning that it has a CPU with a processing (ALU, registers) and control (instruction register, program counter) unit, memory (i.e. RAM), disk storage, and input/output devices (e.g. keyboard, screen).

The architecture has two 16 bit registers (D and A) that can be used by the programmer.
D is used to store values (i.e. data register) while A is both a data register and an address register (meaning that the value in register A can be interpreted as an address in memory).

The ISA of the Hack computer architecture has two types of instructions: the A instruction and the C instruction.
The A instruction is used to set the A register.
Since an instruction is 16 bits on this architecture and the first bit is set to 0 to denote an A instruction, we have 15 remaining bits to represent a value to set the value in the A register to.

The C instruction performs computations, stores the result of those computations somewhere, and decided where to jump next (instruction-wise).
The first bit is a 1 to denote that the instruction is a C instruction.
Then the remaining bits are divided into the computation, the destination, and the jump.
The computation is 7 bits long, the destination is 3 bits long, and the jump is 3 bits long. 
Each different combination of bits for each of these parts correspond to something different.
For example one combination of bits for the computation might indicate that we want to add the values in D and A while another might indicate that we want to AND their values.
Then a particular combination of the destination bits might tell us to store the result in the A register, D register, somewhere in memory, or a combination.
Then a combination of bits in the JMP mnight tell the CPU to do an unconditional jump to whatever value is stored in the A register.
Note that A can be used for deciding wheter to cause a JUMP or as a data memory location for an instruction so well written machine code should make sure that the two are never used in the same program.

### How does an assembler turn symbolic assembly into machine code

The assembler is at its core a text processing program that loops through the mnemonic instructions in the assembly and resolves all symbolic referneces to any addresses.
A symbol table is typicaly used to store and get addresses corresponding to symbols.
The output result is the binary code of the entire program.

## CPU

Now that we know what machine code looks like and how we got machine code from the symbolic assembly (which came from the virtual machine of the compiler), how does the machine code actually get run?

### What does a CPU do

### What are the main components




## Storing Data with Sequential Chips

### How is data stored




## ALU Computations

### How does an ALU perform computations





## Transistors

### How do transistors work

## Conclusion
We conclude our post at the transistor.

## Resources
- [The Elements of Computing Systems](https://www.amazon.com/Elements-Computing-Systems-Building-Principles/dp/0262640686)
- [Computer Systems: A Programmer's Perspective](https://www.amazon.com/Computer-Systems-Programmers-Perspective-3rd/dp/013409266X)
- [Modern Processor Design: Fundamentals of Superscalar Processors](https://www.amazon.com/Modern-Processor-Design-Fundamentals-Superscalar/dp/1478607831)