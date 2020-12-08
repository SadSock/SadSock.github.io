---
layout: post
title:  "为LLVM添加简易RISCV后端(三)：创建后端"
toc: true
date:   2020-12-06 13:30:38 +0800
categories: LLVM
keywords: LLVM
description: 无
---


为一个新的指令集编写编译器是一件复杂的事情，尽管LLVM的出现使这个过程比之前简单多了！
一个匪夷所思的困难是缺乏一个简单的、循序渐进的教程[^1][^2]。
因此，本系列博客试图提供一个从头开始编写LLVM后端的简易教程来解决（部分）问题。


## 创建后端

开发LLVM后端并不是一件特别吸引人的事情。
您很快就会意识到，这项工作在很大程度上就是从其他现有后端复制代码。
在线论坛上，LLVM开发者建议从“复制一个现有的后端，重命名并修改它以适应您的需要”开。
但是即使是相对较小的后端，比如Lanai或XCore，也相当复杂，而且代码也不容易理解！

在本系列文章中，将采取略有不同的方法。
我们将使用现有的LLVM后端作为起点，但我已经删除了大部分代码，并将其减少到编译一个(很小的)程序所需的最低限度。
精简的后端，称为RISCW，非常简单，可以帮助理解LLVM目标独立代码生成器，而不必纠缠于细节。
在这篇文章的其余部分，我将使用RISCW后端来展示如何创建一个新的LLVM后端。
我们还将看到如何用一个实验性的后端构建LLVM，甚至编译一个(非常简单的) C程序到汇编。


### Triple和ELF配置

我们首先为后端配置一个新的目标描述Triple。
由于历史原因，Triple编码了目标平台的重要信息（如体系结构、供应商和操作系统）。
以下是配置一个新的Triple的步骤:

1. 在llvm/include/llvm/ADT/Triple.h用Triple声明一个新的体系结构 (见这里)。
2. 提供字符串和Triple之间的类型转换(参见这里,这里及这里)。
3. 指出后端支持的目标文件类型，例如ELF、COFF等，ricw只支持ELF(参见这里)。
4. 指出目标平台的字平台，例如32位或64位，以及指针的大小(请参阅这里,这里及这里)。

**注意:**你可以在这里和这里找到更多关于Triple的信息。


**注意:**指令集并不一定意味着指针的大小。
例如，在为RV64编译时，指针并不总是64位的。
指针大小通常由ABI给出，在64位机器中，它可以是ilp32(即int、long和指针为32位)。

下面的参数用于配置ELF:

1. 创建一个枚举值作为RISCW的体系结构的标识(见此处)。
这个值被编码在ELF文件头的e_machine字段中。
这个值不是随意设置的; 它必须取得授权，例如:0xF3 for RISCV。
但是我们现在将它设置为一个未使用的值。

2. 声明ELF重定位类型(见这里和这里)。
同样，这些是依赖于架构的，这里列出了用于RISCV的类型。
在这个阶段，我们将简单地为RISCW放置了占位符。

3. 文件格式名称(见此处)。

4. 指示给定类的目标描述Triple(见此处)。目前，ELF头中的类是一个字节，用于对格式是32位还是64位进行编码。


**注意:**查看wikipedia获取更多关于ELF文件的信息。



### 配置驱动器

回想一下，我们使用clang将输入的C代码编译成LLVM IR。
但是clang不仅仅是我们的编译器前端，它也是一个驱动器，类似GCC，驱动编译流水线将输入的C程序转换为另一个表示，比如把C转换为汇编或目标代码。
因此，我们需要告诉clang

1. 新后端的支持特性。例如，clang需要知道RISCW是32位还是64位。

2. 新后端的编译流程。例如，它应该使用什么汇编程序? 什么连接器? 有哪些包括路径等等。


我们可以通过添加一个新的target类`RISCWTargetInfo`来告诉clang有关RISCW的信息，该类与LLVM已有的target类一起被实例化，如这里所示。
该类在这里和这里分别被声明和定义。
在这段代码中有一些重要的事情需要强调:

* `RISCWTargetInfo`通过字符串描述数据布局。这个字符串编码许多重要信息，比如指针中每一位、堆栈对齐要求等。
* 基本C数据类型的大小。
* 函数`RISCWTargetInfo::getTargetDefines(**`指示编译时定义的C预处理器宏，例如，这些宏是在使用RISCV后端编译代码时定义的。
宏通常描述后端支持的体系结构、ABI、启用/禁用任何特性等

**注意:**一个后端可能支持多个指令集和ABI，因此驱动器的配置必须根据选定的目标Triple进行更改。
例如，`RISCWTargetInfo`根据Triple包含riscv32还是riscv64来更改数据布局字符串。

**注意:**这里可以查看RISCWTargetInfo的父类TargetInfo的声明。
它包含了更多的可以配置的选项。

配置工具链相对简单。
我们只需要实现一个从Toolchain继承的RISCWToolChain类，如下所示。
代码基本上是不言自明的，通过覆盖ToolChain类的成员，您可以修改更多的选项(见此处)。

### 创建新Target

每个后端在llvm/lib/Target下都有一个单独的目录，其中包含后端的大部分代码。
我们不会在这篇文章中深入讨论代码的细节(稍后我们会这样做) ，因为即使是一个很小的后端，比如RISCW，也有很多文件。
目前，我们可以将这些文件大致分为三类:

* **TableGen文件**LLVM目标无关代码生成框架实现了一个精心设计的模式匹配算法，用于为输入的程序选择指令。
待匹配的模式使用TableGen语法描述。
此外，TableGen文件还描述了target在体系结构方面的重要特性，如寄存器的数量和调用约定等。

* **Build文件**后端的每个目录都必须被声明，否则它将不会被构建。
此外，我们的后端的顶部目录(`llvm/lib/Target/RISCW`) ，以及每个子目录必须包含两个构建文件:`CMakeLists.txt`和`LLVMBuild.txt`，
前者将源文件和任何子目录添加为生成目标，而后者为生成目标设置简单的生成参数，参数包括生成目标的名称、链接所需的库等。

* **C++文件**包含了大量的后端代码，实现了从简单的配置选项到更复杂的指令选择功能(TableGen没有实现或不能实现)的所有功能。

### 建立实验性后端

现在，一切都已经建立，我们可以构建带有RISCW后端的LLVM。
但是我们不能简单地根据上一章的内容修改CMake的`-DLLVM_TARGETS_TO_BUILD`选项，以包含RISCW，因为后端仍处于试验阶段。
相反，我们使用`-DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD`选项，如下:
```sh
cmake -G "Ninja" -DLLVM_ENABLE_PROJECTS="clang" -DLLVM_TARGETS_TO_BUILD="ARM;Lanai;RISCV" -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="RISCW" -DCMAKE_BUILD_TYPE="Debug" -DLLVM_ENABLE_ASSERTIONS=On ../llvm
ninja
```
当构建完成后，你可以检查RISCW现在是否是一个可用的后端，如下所示:
```sh
$ ./build/bin/llc --version
LLVM (http://llvm.org/):
  LLVM version 10.0.1
  DEBUG build with assertions.
  Default target: x86_64-unknown-linux-gnu
  Host CPU: znver2

  Registered Targets:
    arm     - ARM
    armeb   - ARM (big endian)
    lanai   - Lanai
    riscv32 - 32-bit RISC-V
    riscv64 - 64-bit RISC-V
    riscw   - 32-bit RISC-V         <== YAY!!
    thumb   - Thumb
    thumbeb - Thumb (big endian)
```
### 编译C程序
我们的RISCW后端只能发出两条add和ret指令，而且它不能正确处理函数调用、堆栈和几乎所有其他的东西！
因此，我们将约束自己，只编译这个小函数:
```sh
int test(int a, int b)
{
    return a + b;
}
```
就这样，我们得到了这样一个代码:
```sh
	.text
	.file	"test.c"
	.globl	test                    ; -- Begin function test
	.type	test,@function
test:                                   ; @test
; %bb.0:                                ; %entry
	add	x0, x1, x0
	ret
.Lfunc_end0:
	.size	test, .Lfunc_end0-test
                                        ; -- End function
	.ident	"clang version 10.0.1 (https://github.com/llvm/llvm-project 89f2d2cc3bba7cb12cee346b3205cb0335e758cd)"
	.section	".note.GNU-stack","",@progbits
```
有很多东西缺失了，代码实际上是不正确的，在RISCV中的x0是一个硬编码为0的只读寄存器。
但是我认为我们已经达到了目标: 建立了一个最小的LLVM后端，可以很容易地用更多的特性进行扩展。

**注意:**如果您使用上一篇文章中的命令来编译上面的测试函数，请确保为clang设置了`-target riscw`和为llc设置了`-march=riscw`。

**注意:**试图编译更复杂的程序将导致```cannot select...```错误。如果你感兴趣，就试一试。

**注意:**您可以通过将`-debug`选项传递给`llc`来指示编译器打印调试信息。


## 注释
[^1]:公平地说，有不少关于LLVM的书籍和网站，但大多数都是对这个工具的一般性描述，还有是关于如何编写新前端的实践教程，但后端的教程非常少。
[^2]:[这个教程](https://jonathan2251.github.io/lbd/)描述了如何开发LLVM后端，但我发现很难理解。
