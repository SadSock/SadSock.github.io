---
layout: post
title:  "为LLVM添加简易RISCV后端(二)：入门"
toc: true
date:   2020-12-04 19:30:38 +0800
categories: LLVM
keywords: LLVM
description: 无
---

为一个新的指令集编写编译器是一件复杂的事情，尽管LLVM的出现使这个过程比之前简单多了！
一个匪夷所思的困难是缺乏一个简单的、循序渐进的教程[^1][^2]。
因此，本系列博客试图提供一个从头开始编写LLVM后端的简易教程来解决（部分）问题。


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
