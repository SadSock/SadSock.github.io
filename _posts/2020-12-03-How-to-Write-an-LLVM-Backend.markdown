---
layout: post
title:  "How to Write an LLVM Backend"
toc: true
date:   2020-12-03 19:30:38 +0800
categories: LLVM
keywords: LLVM
description: 
---

为一个新的指令集编写编译器是一件复杂的事情，尽管LLVM的出现使这个过程比之前简单多了！
一个匪夷所思的困难是缺乏一个简单的、循序渐进的教程[^1][^2]。
因此，本系列博客试图提供一个从头开始编写LLVM后端的简易教程来解决（部分）问题。


## 简介


在本教程中，我将为RISC V指令集的基本32位版本（即RV32IM）开发一个后端。希望这能帮助那些不熟悉LLVM的人开始使用这个工具，并将其扩展到自己的项目中。看懂本教程不需要前置知识，但是如果你熟悉C++和RISC V，学习本教程会更容易。

在本文剩下的部分中，我将简要描述LLVM的体系结构和后端结构。不过，我不会在这里详细说明，因为，如果你像我一样，在5分钟（或5秒）之后，就会忘记读过的任何繁琐文档。LLVM的细节将在以后的帖子中根据需要提供。

**注意:**如果你想要更详细的版本，你可以自娱自乐地阅读[LLVM User Guides](http://llvm.org/docs/UserGuides.html)。

### LLVM架构

传统上，编译过程分为三个阶段。
首先，编译器的前端将源代码转换为某种中间表示（IR）；
然后，优化IR；
最后，编译器的后端将IR转换为机器代码。
传统的编译器通常仅支持一种编程语言和一种目标指令集，编译器的源代码很难重用，例如添加新的目标指令集。

LLVM模块化地实现了三个编译过程，可以解决重用问题。
其思想是LLVM的核心（即IR和优化器）是固定的，但是前端和后端可以被替换，以使编译器可以支持多种编程语言和指令集。
例如，我们可以使用Clang（LLVM的前端）和x86后端把C/C++代码编译成X86指令集上的可执行程序。
我们也可以用ARM后端替换X86后端，从而得到ARM指令集上的可执行程序。

**注意：**LLVM的设计师Chris Lattner撰写了这篇[文章](http://www.aosabook.org/en/llvm.html)介绍LLVM的体系结构及其设计动机。

**注意：**LLVM的这三个阶段的每个阶段在都有一个专用的可执行文件。
clang是C/C++的前端（显然针对不同的编程语言有不同的前端），opt是优化程序，llc用于调用后端。
通常，我们使用clang作为驱动程序来执行前端，使用llc和opt配合适当的参数来生成IR，汇编，可执行文件等。


### 代码生成
LLVM后端将IR编译为目标代码或汇编代码。
每个后端都只支持单一平台，但可以支持多个指令集。
例如，LLVM只有一个ARM后端，但该后端可以为ARMv6和ARMv7等指令集生成代码。
每个后端都建立在LLVM的目标无关代码生成器之上。
目标无关代码生成器是一个框架，可实现诸如寄存器分配之类的关键算法。
从广义上讲，后端的任务是配置该框架并使之适应其目标指令集的特定需求。

代码生成具有以下阶段：

1. **指令选择** 映射LLVM IR到目标指令集中的指令。
此阶段使用无限数量的虚拟寄存器和函数调用堆栈的抽象引用。

2. **计划和编排** 确定指令的顺序。
需要明确的是，在指令选择阶段已经对指令进行了排序，但这里可以根据寄存器分配策略或指令等待时间来对其中的一些指令的排序进行优化。

3. **基于SSA的机器代码优化** 执行诸如[peephole](https://en.wikipedia.org/wiki/Peephole_optimization)优化之类的工作。

4. **寄存器分配** 将虚拟寄存器映射到物理寄存器。

5. **Prolog / Epilog插入** 在每个函数的开头（或prolog）和结尾（或epilog）插入机器指令。
这些通常是在进入或退出函数时扩展堆栈的指令。
由于当前已经知道堆栈大小，因此也可以解析抽象堆栈引用。

6. **后期机器代码优化** 可能不言自明。

7. **代码发射** 发出目标代码或汇编代码。

接下来，我将看一下构建LLVM以及如何设置开发/调试环境…

**注意：**您可以阅读[这篇文章](http://llvm.org/docs/CodeGenerator.html)了解LLVM目标无关代码生成器的更多信息。


## 入门

在为新项目编写代码之前，我通常会配置环境，并对查看经存在的代码，这就是这一节要做的。在这一节中，我将展示如何下载编译LLVM和其他对调试有用的工具。我们还将了解如何使用现有的LLVM后端和GNU工具链来编译、汇编、链接和运行程序。

### 环境

我正在使用Ubuntu，但是你应该能够在其他系统中重复这些步骤，而且(相对来说)几乎没有什么不同。您将需要以下工具来构建软件。

* Makefile
* C/C++ Compiler – 我用 GCC 9.2.1
* autotools
* CMake
* Ninja
* Git
* 大量耐心

**注意：**我可能忘记了一些东西，但是构建系统会通过一个错误告诉您；

### 编译LLVM

LLVM维护者已经建立了这个方便的repo，它包含LLVM和工具链的其他部分，比如Clang。

```
git clone https://github.com/llvm/llvm-project
```

在本系列文章中，我们将使用llvm 10.0.1，我建议您也使用该版本的LLVM。
因为LLVM的变化非常快，这里显示的一些代码在旧/新版本中可能无法工作。
不过，原理应该大致相同。

LLVM使用CMake为构建系统生成构建文件，LLVM支持的构建系有：Ninja，Makefiles，Visual Studio和XCode。
我通常使用Ninja，因为我认为它在我的系统中速度最快（我没有证据支持该判断！）。
您可以通过cmake命令的`-G`参数来更改构建系统。

CMake有很多选项，我鼓励您对其进行研究，因为有些选项对调试非常有帮助。
您可以在[这里](https://llvm.org/docs/CMake.html)阅读所有构建选项。
在本教程中，我将使用以下选项:

1. `-DLLVM_ENABLE_PROJECTS` 构建编译器的其余部分，比如Clang。

2. `-DLLVM_TARGETS_TO_BUILD` 指定要构建的后端。查看其他后端的输出对调试很有帮助，但是如果添加太多，构建会花费很长时间。

3. `-DCMAKE_BUILD_TYPE` 构建Debug版本。

4. `-DLLVM_ENABLE_ASSERTIONS=On` 启用断言，对调试很有帮助。


以下是在克隆repo之后构建LLVM的方法。
```sh
cd llvm-project
git checkout llvmorg-10.0.1
mkdir build
cd build
cmake -G "Ninja" -DLLVM_ENABLE_PROJECTS="clang" -DLLVM_TARGETS_TO_BUILD="ARM;Lanai;RISCV" -DCMAKE_BUILD_TYPE="Debug" -DLLVM_ENABLE_ASSERTIONS=On ../llvm
ninja
```

**注意：**您可以在[这里](https://llvm.org/docs/GettingStarted.html)和[这里](https://llvm.org/docs/CMake.html)找到更多有关构建LLVM的信息。

**注意：**您可以为Ninja传递`-j <NUM_JOBS>`选项，以指示要并行的作业数。 
过高的`<NUM_JOBS>`会导致构建崩溃，并产生`collect2：ld ...`错误消息。


### 编译RISC V的GNU工具链

你可能有点困惑，为什么我建议构建GCC的RISC V后端？
难道我们不是要自己编写编译器后端吗？

我们构建GCC的RISC V后端，是因为我们希望在初始阶段使用GCC的汇编器和链接器来测试LLVM后端生成的代码。
编译过程分为很多阶段，在初始阶段，我们已经有以下结构:

* Clang 编译C代码到LLVM IR
* LLVM 优化IR
* LLVM后端 编译IR到汇编
* GCC 汇编和链接可执行文件

使用以下命令下载，构建和安装GCC for RISCV。

```bash
git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
mkdir build
cd build
../configure --with-arch=rv32gc --with-abi=ilp32
make
make install
```
**注意：**请确保为指令集的正确变体（即RV32）构建GCC工具链，因为构建系统的默认值为RV64！

**注意：**GNU工具链支持RISC V的多个ABI，例如`ilp32`，`ilp32d`和`ilp32f`，这取决于您是否需要软浮点，硬浮点。



### 编译C程序

现在，构建和运行C代码的环境已经配置好了，尽管我们还没自己的后端（还！）。让我们从一个简单的C程序开始：
```C++
#include <stdio.h>

int main(void)
{
    printf("Hello world!\n");
    return 0;
}
```

首先，使用Clang将C代码编译为LLVM IR。
我们的计划是使用标准库中来自头文件stdio.h的函数printf，如果不能找到头文件，编译器会提示出错。
为了使用GCC自带的RISC V标准C库，我们使用了`-isystem`参数。
这会将包含所需头文件的目录添加到Clang预处理器的搜索目录列表中。

```sh
clang -O2 -emit-llvm -target riscv64 -isystem <PATH_TO_GCC>/riscv64-unknown-elf/include -c test.c -o test.bc
```
上面的命令把C语言文件test.c编译到LLVM IR文件test.bc，这是专门为机器设计的语言人类很难直接阅读。
我们可以使用以下命令反汇编该文件：
```sh
llvm-dis test.bc
```
现在，使用包含以下内容的后端将IR编译为程序集，而无需使用以下命令下载LLVM：
现在，使用LLVM自带的后端将IR编译为程汇编：
```sh
llc -march=riscv64 -O2 -filetype=asm test.bc -o test.S
```
GCC可以直接生成程序的二进制文件。
我将其分为两个步骤，但是您可以根据需要使用单个命令。
```sh
riscv64-unknown-elf-gcc -c test.S -o test.o
riscv64-unknown-elf-gcc test.o -o test
```
最后，我们可以使用模拟器或真实硬件运行程序。








## 注释
[^1]:公平地说，有不少关于LLVM的书籍和网站，但大多数都是对这个工具的一般性描述，还有是关于如何编写新前端的实践教程，但后端的教程非常少。
[^2]:[这个教程](https://jonathan2251.github.io/lbd/)描述了如何开发LLVM后端，但我发现很难理解。
