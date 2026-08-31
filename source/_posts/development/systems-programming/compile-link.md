---
title: C/C++ 编译与链接
date: 2026-09-01 01:47:37
tags:
  - c
  - cpp
  - compiler
  - linker
  - gcc
categories:
  - Development
  - Systems Programming
---


编译与链接主要为下述步骤，其中前三个步骤就是广义上的编译。

1. 预处理：把一个 `.c` 源文件处理成 `.i` 预处理文件
2. 编译：把 `.i` 预处理文件进一步处理成 `.s` 汇编文件（狭义上的编译）
3. 汇编：把 `.s` 汇编文件最终处理得到 `.o` 机器码文件
4. 链接：把多个 `.o` 机器码文件链接成可执行文件

# 准备源代码

为了演示这个过程，我编写了以下两个文件作为源代码。

```c
// myheader.h
#define N 666
#define M 999
```

```c
// main.c

#include "myheader.h"

/**
 * 我是注释
 * 我也是注释
 */

int main() {
  // 我还是注释
  int i = M + N;
  return 0;
}
```

接下来，我将以此文件为基础进行演示。

# 编译（广义上的编译）

广义上的编译指的是从高级编程语言的代码到二进制机器码的过程。

## 预处理

预处理会对 `#` 开头的预处理语句进行处理。

通过 `gcc -E main.c -o main.i` 命令将 main.c 预处理成 main.i。

main.i 文件内容如下：

```c
# 0 "main.c"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 0 "<command-line>" 2
# 1 "main.c"
# 1 "myheader.h" 1
# 2 "main.c" 2








int main() {

  int i = 999 + 666;
  return 0;
}
```

可以看得出来 `#include` 和 `#define` 这些预处理指令，都已经成功地执行。

- `#include` 成功地把 `myheader.h` 引入到了 `main.c` 中
- `#define` 的宏定义也完成了文本替换
- 源代码中的注释也全部被去掉了


## 编译（狭义上的编译）

狭义上的编译指的是把预处理文件编译成汇编代码的过程。

通过 `gcc -S main.i -o main.s` 命令将 main.i 编译成 main.s。

main.s 文件内容如下：

```asm
	.file	"main.c"
	.text
	.globl	main
	.type	main, @function
main:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movl	$1665, -4(%rbp)
	movl	$0, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE0:
	.size	main, .-main
	.ident	"GCC: (GNU) 14.2.1 20240910"
	.section	.note.GNU-stack,"",@progbits
```

可以看得出来预处理好的文件已经被编译成了汇编代码。

懂一些汇编的话，应该能看出来 `movl $1665, -4(%rbp)` 这一行汇编代码就是给变量 i 赋值，rbp 寄存器偏移 -4 的位置就是 i 的地址。


## 汇编

汇编指的是从汇编代码到机器码这个过程，汇编代码和机器码指令是一一对应的。

通过 `gcc -c main.s -o main.o` 命令将 main.s 汇编成 main.o。

main.o 就是二进制机器码文件，由于汇编代码和机器码一一对应，我们可以从汇编代码知道 main.o 中暴露了 `main` 这个符号，并且标记了 `main` 是一个函数。在后续链接的时候，因为 C 语言中 main 函数的特殊性，它将会被处理为整个二进制程序的入口。

如果我们的代码是作为库提供给其他人使用的话，可能就不会有 main 函数，也不会进入下面的链接环节，而是把多个 `.o` 文件打包成一个文件，这个打包好的文件也就是库文件，根据打包过程的不同可以分为动态库和静态库。



# 链接

最后我们可以使用 `gcc main.o -o main` 将编译获得的多个 `.o` 文件链接成为可执行程序。在本文的例子中，只有一个 `.o` 文件。

最终 `main` 就是可执行程序。



# 目录结构

```text
test/
├── main
├── main.c
├── main.i
├── main.s
├── main.o
└── myheader.h
```

