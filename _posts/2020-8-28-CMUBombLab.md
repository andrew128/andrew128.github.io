---
layout: post
title: Defusing CMU's Bomb Lab using GDB (8/28/20)
---
This post walks through CMU's 'bomb' lab, which involves defusing a 'bomb' by finding the correct inputs to successive phases in a binary executable using GDB.

## Post Outline
- [Intro](#intro)
- [GDB](#gdb)
- [Phase 1](#phase-1)
- [Phase 2](#phase-2)
- [Phase 3](#phase-3)
- [Resources](#resources)

## Intro
This post walks through the first 3 phases of the lab.
I used a linux machine running x86_64.
Also note that the binary follow the AT&T standard so instruction operations are reversed (e.g. mov a b moves data from a to b as opposed to b to a).

## GDB
Here are a few useful commands that are worth highlighting:
#### layout asm 
This command divides the screen into two parts: the command console and a graphical view of the assembly code as you step through it. 
Control-l can be used to refresh the UI whenever it inevitably becomes distorted.
While layout asm is helpful, also helpful to view the complete disassembled binary.
We can get the full assembly code using an object dump: `objdump -d path/to/binary > temp.txt`.

#### i r
This command lists out all the values that each of the registers hold. 

#### i b
This command lists all the current breakpoints as well as how many times each breakpoint has been hit on the current run.

#### b *memory address, function name
This command sets breakpoints throughout the code.
Breakpoints can be set at specific memory addresses, the start of functions, and line numbers. 

#### x memory address, register
This command prints data stored at a register or memory address.
It is useful to check the values of these registers before/after entering a function.
For example, after a function has finished executing, this command can be used to check the value of $rax to see the function output.

## Phase 1

We can then set up a breakpoint upon entering `phase_1` using `b phase_1` and for the function `explode_bomb` to avoid losing points.
Using `layout asm`, we can see the assembly code as we step through the program.
Let's enter the string `blah` as our input to `phase_1`.
When we hit `phase_1`, we can see the following code:
```
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp // allocate space on the stack
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi // move data from some memory to esi register
  400ee9:	e8 4a 04 00 00       	callq  401338 <strings_not_equal> // call a function
  400eee:	85 c0                	test   %eax,%eax 
  400ef0:	74 05                	je     400ef7 <phase_1+0x17> // if eax is equal, avoided the bomb
  400ef2:	e8 43 05 00 00       	callq  40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	retq   
```

The code is annotated with comments describing each line.
We see that a `strings_not_equal` function is being called.
From this, we can guess that to pass `phase_1`, we need to enter the correct string.

Let's set a breakpoint at `strings_not_equal`.
Once we enter the function, we can check the registers that store the first two inputs: $rdi and $rsi.

![_config.yml]({{ site.baseurl }}/images/CMUBombLab/p1.png)

We can see that our string input `blah` is being compared with the string `Border relations with Canada have never been better.`.
Entering this string defuses phase_1.

## Phase 2
Let's clear all our previous breakpoints and set a new one at `phase_2`.
Let's use `blah` again as out input for `phase_2`.

We can now see the assembly code.
Each line is annotated.
```
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp
  400f02:	48 89 e6             	mov    %rsp,%rsi
  400f05:	e8 52 05 00 00       	callq  40145c <read_six_numbers> // read 6 numbers
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp) // check to see that first # is 1
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	callq  40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax // move the previous number into %eax
  400f1a:	01 c0                	add    %eax,%eax // double %eax
  400f1c:	39 03                	cmp    %eax,(%rbx) // compare %eax and the current number
  400f1e:	74 05                	je     400f25 <phase_2+0x29> // if equal, avoid bomb
  400f20:	e8 15 05 00 00       	callq  40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx
  400f29:	48 39 eb             	cmp    %rbp,%rbx
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b> // jump back to 400f17
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40> // otherwise end of loop, exit
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	retq 
```

From the code, we can see that we first read in 6 numbers.
To see the format of how we enter the six numbers, let's set a breakpoint at `read_six_numbers`.

```
000000000040145c <read_six_numbers>:
  40145c:	48 83 ec 18          	sub    $0x18,%rsp
  401460:	48 89 f2             	mov    %rsi,%rdx
  401463:	48 8d 4e 04          	lea    0x4(%rsi),%rcx
  401467:	48 8d 46 14          	lea    0x14(%rsi),%rax
  40146b:	48 89 44 24 08       	mov    %rax,0x8(%rsp)
  401470:	48 8d 46 10          	lea    0x10(%rsi),%rax
  401474:	48 89 04 24          	mov    %rax,(%rsp)
  401478:	4c 8d 4e 0c          	lea    0xc(%rsi),%r9
  40147c:	4c 8d 46 08          	lea    0x8(%rsi),%r8
  401480:	be c3 25 40 00       	mov    $0x4025c3,%esi // Move some value into a register
  401485:	b8 00 00 00 00       	mov    $0x0,%eax
  40148a:	e8 61 f7 ff ff       	callq  400bf0 <__isoc99_sscanf@plt> // scanf() is called
  40148f:	83 f8 05             	cmp    $0x5,%eax
  401492:	7f 05                	jg     401499 <read_six_numbers+0x3d>
  401494:	e8 a1 ff ff ff       	callq  40143a <explode_bomb>
  401499:	48 83 c4 18          	add    $0x18,%rsp
  40149d:	c3                   	retq   
```

From the above, we see that we are passing some value into a register before calling `scanf()`. 
Knowing that `scanf()` takes in a string format as its input, let's break right before `scanf()` is called and check the value of $esi.
```
(gdb) x /s $esi
0x4025c3:       "%d %d %d %d %d %d"
```
From this, we can see that the input format of `read_six_numbers` should be 6 space-separated integers.

Going back to the code for `phase_2`, we see that the first number has to be `1`.
```
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp) // check to see that first # is 1
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	callq  40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
```

We then move onto the next few lines.
```
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax // move the previous number into %eax
  400f1a:	01 c0                	add    %eax,%eax // double %eax
  400f1c:	39 03                	cmp    %eax,(%rbx) // compare %eax and the current number
  400f1e:	74 05                	je     400f25 <phase_2+0x29> // if equal, avoid bomb
  400f20:	e8 15 05 00 00       	callq  40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx
  400f29:	48 39 eb             	cmp    %rbp,%rbx
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b> // jump back to 400f17
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40> // otherwise end of loop, exit
```

From the above annotations, we can see that there is a loop.
At each iteration, we check to see that the current value is double the previous value.
From this, we can deduce that the input for `phase_2` should be `1 2 4 8 16 32`.

## Phase 3
Let's now set a breakpoint at `phase_3`.
The following lines are annotated.

```
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi // reading $0x4025cf results in "%d %d"
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	callq  400bf0 <__isoc99_sscanf@plt> // call scanf()
  400f60:	83 f8 01             	cmp    $0x1,%eax
  400f63:	7f 05                	jg     400f6a <phase_3+0x27> // if output greater than 1 (read more than 1 number), avoid bomb
  400f65:	e8 d0 04 00 00       	callq  40143a <explode_bomb>
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp) // Compare out first number 0x8(%rsp) with 0x7
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax
  400f75:	ff 24 c5 70 24 40 00 	jmpq   *0x402470(,%rax,8)
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb>
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax // compare our second input 0xc(%rsp) with value in %eax (682)
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	callq  40143a <explode_bomb>
  400fc9:	48 83 c4 18          	add    $0x18,%rsp
  400fcd:	c3                   	retq  
``` 

From the above comments, we deduce that we want to input two space-separated integers.
The first number we can try to be 6 and the second must be 682.
Entering these numbers allows us to pass `phase_3`.

## Resources
- [CMU Lab Website](http://csapp.cs.cmu.edu/3e/labs.html)