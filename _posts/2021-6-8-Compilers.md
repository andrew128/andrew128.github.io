---
layout: post
title: An Introduction to Compilers
---
This blog post introduces the world of compilers with an example in Golang.
The code comes from the book [Writing An Compiler In Go](https://compilerbook.com/) by Thorsten Ball.

## Post Outline
- [Introduction](#introduction)
- [Background]
    - [How do compilers work]()
    - [How do compilers compare to interpreters]()
    - [What about languages that are both compiled and interpreted]()
    - [Virtual machines]()
    - [The compiler in the book]()
- [Concrete Code]()
- [Resources]

## Introduction
In this blog post we will be covering the main parts of writing a compiler.
The post is divided into two main parts: background (where I go over the basic outline and details of compilers) and code (where I bring up specific examples from the book Writing a Compiler in Go).

## Background

### How do compilers work

In general, compilers take in source code and output machine code that can be executed on a specific architecture.
More specifically, compilers take in source code and parse it into an AST (abstract syntax tree as covered in a [previous blog post](https://andrew128.github.io/Interpreters/)) using a lexer and a parser.
The AST is then converted into an intermediate representation upon which several optimization passes are applied to.
Then a code generator outputs the target machine code.
There are many more steps involved but these ones cover the major parts.

INCLUDE PICTURE OF GENERAL PROCESS

### How do compilers compare to interpreters

Interpreters, like compilers, are another common kind of language processor.
Both will have the same front end where the source code is lexed and parsed into an abstract syntax tree (AST).
After the AST is where the paths diverge.
An interpreter directly executes the operations specified in the source program on user supplied inputs
A compiler, on the other hand, produces a target program in the target language that is executed when the user supplied inputs.

A compiler-produced program is typically much faster than an interpreter because the machine code from the compiler program is optimized for the specific architecture. 
On the other hand, an interpreter can be much better at providing error messages that are language specific to the source code because there isn't the source/target language abstraction layer that the compiler has.

### What about languages that are both compiled and interpreted

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

There is also a memory region called the call stack, or just the stack.
This region is where the CPU performs LIFO (last in first out) operations on.
The call stack holds the return address, local variables, and function arguments.
A stack data structure is chosen because it plays well with nested function calls.
The pushing and popping of instructions fit how functions call one another before returning to their caller.
[This blog post](https://andrew128.github.io/x86-binary/) goes into more detail about the calling conventions, specifically on the x86 architecture.

CPUs also have processor registers to store data.
This is a step up from the memory hierarchy from RAM.
It is much faster but has significantly less storage space, much like how RAM compares to disk.
In an x86-64 architecture, the CPU has 16 general purpose registers holding 64 bits of data each.
Registers are used for data that is used very frequently, such as a pointer to the next instruction to execute in the stack (i.e. the stack pointer).

#### Physical Machine to Virtual Machine

Now that we know what the virtual machine is emulating, lets see how the virtual machine mimics the hardware. 
The virtual machine has a fetch decode execute cycle just like the CPU.
It also has a program counter that points to the next instruction to execute.
There is also a call stack and multiple registers.

An example of a processor virtual machine is the Java Virtual Machine (JVM).

#### Stack based VM vs Register based VM

There are two broad categories of virtual machines: stack based virtual machines and register based virtual machines.

A stack based virtual machine is a virtual machine that relies completely on its call stack to perform all operations.
A register based virtual machine is a virtual machine that has a call stack and registers.
Instructions can use the registers in addition to the stack.
This results in less instructions compared to the stack based VM because we don't need to push/pop everytime to perform a computation.
However, a register based machine is more complicated to implement simply because it has more pieces (i.e. the registers) in addition to the call stack.

### The compiler in the book
The code is from the book's `Monkey` language and is written in Golang.
The compiler uses a stack based virtual machine.

## Concrete Code

The following sections show concrete code from the compiler.

### Bytecode

The code shown is for a compiler using a stack based virtual machine.
As mentioned previously, source code is compiled into bytecode and interpreted by the virtual machine to be executed.
Bytecode can then be thought of as instructions that tell the virtual machine what to do.

The instructions, represented by opcodes, in the bytecode are each one byte in size.
In addition, the operands (parameters to the opcodes) are also represented by bytes.

We can see this in the `Opcode` and `Instructions` definitions in `Monkey`.

```
type Instructions []byte
type Opcode byte
```

// DEFINITION

Now that we've defined the type of an Opcode, lets see how actual opcodes are defined.
The OpCode definitions are a map with an OpCode byte being the key and the definition being the value.
A Definition type is defined to eb made up of the name of the operand (e.g. PUSH) and the number of bytes each operand takes up in OperandWidths.

```
type Definition struct {
	Name          string
	OperandWidths []int
}
```

For instance, the OpConstant operator, representing an integer, can be added to the definitions map like so:

```
var definitions = map[Opcode]*Definition{
    ...
	OpConstant: {"OpConstant", []int{2}},
    ...
}
```

The operand is 2 bytes wide, indicating that it is of type uint16.

Based off a given OpCode, there will be a function that creates the corresponding bytecode.
The function `Make()` takes in an opcode and its corresponding operands and returns a byte array representing the bytecode.
The function works by looking up the opcode in the definition.
It then creates a byte array where the first value is the opcode and the rest are any operands.

```
func Make(op Opcode, operands ...int) []byte {
	def, ok := definitions[op]
	if !ok {
		return []byte{}
	}

	instructionLen := 1
	for _, w := range def.OperandWidths {
		instructionLen += w
	}

	instruction := make([]byte, instructionLen)
	instruction[0] = byte(op)

	offset := 1
	for i, o := range operands {
		width := def.OperandWidths[i]
		switch width {
		case 2:
			binary.BigEndian.PutUint16(instruction[offset:], uint16(o))
		case 1:
			instruction[offset] = byte(o)
		}
		offset += width
	}

	return instruction
}
```

#### Endianess
Note that we can order the bytecode in one of two ways: left to right or right to left in terms of least significant to most significant bit.
Different architectures order the bytecode in one of the two ways.

### Compiler and Virtual Machine
So we now know how bytecode is represented and created from opcodes and their operands (i.e. assembly).
The compiler generates the bytecode from the parsed program.
Let's first take a look at repl, which takes in a program, parses it into an AST as covered in the [interpreter blog post](https://andrew128.github.io/Interpreters/), and compiles it.
The bytecode is then passed to the virtual machine, which interprets the bytecode.
This is shown in the following code:
```
func Start(in io.Reader, out io.Writer) {
	scanner := bufio.NewScanner(in)

	for {
		fmt.Printf(PROMPT)
		scanned := scanner.Scan()
		...

		line := scanner.Text()
		l := lexer.New(line)
		p := parser.New(l)

		program := p.ParseProgram()
		...

		comp := compiler.New()
		err := comp.Compile(program)
		...

		machine := vm.New(comp.Bytecode())
		err = machine.Run()
		...

		stackTop := machine.StackTop()
		io.WriteString(out, stackTop.Inspect())
		io.WriteString(out, "\n")
	}
}
```

#### Compiler
- what the compile(Program) function is doing
The compiler has a `Compile(program)` function that parses the AST nodes and emits the respective bytecode.
There is a switch statement for all the possible AST nodes.
An example with a constant is shown below:

```
func (c *Compiler) Compile(node ast.Node) error {
	switch node := node.(type) {
	
    ...

	case *ast.IntegerLiteral:
		integer := &object.Integer{Value: node.Value}
		c.emit(code.OpConstant, c.addConstant(integer))

    ...

	return nil
}
```

The compiler's emit function calls the bytecode `Make()` function (shown previously) and appends the resulting bytes to the compiler's array of bytes `instructions` representing all the bytecode compiled so far.

#### Virtual machine

- the fetch decode execute cycle of the code - vm.go Run()

### Compiling Expressions
- compiling expressions << show code for simple add, subtract, etc.

### Conditionals

### Symbol Table

### Compiling Functions

## Resources
- Writing a Compiler in Go
- Compilers: Principles, Techniques, and Tools
- The Elements of Computing Systems