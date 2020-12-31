---
layout: post
title:  "Write an LLVM Backend(零)：简介"
date:   2020-11-01 18:38:38 +0800
toc: true
categories: LLVM
keywords: LLVM
description: 无
---

# 简介

本文档描述了编写编译器后端的技术，
这些后端将LLVM中间表示（IR）转换为特定机器或其他语言的代码，
这些代码可以是汇编代码也可以是二进制代码（可用于JIT编译器）。

LLVM的后端是一个平台独立的代码生成器，
它可以为不同类型的目标CPU生成代码，
包括X86、PowerPC、ARM和SPARC。
后端还可以为GPU或Cell处理器的SPU生成代码，
以支持计算内核的执行。

本文档主要关注`llvm/lib/Target`目录中的现有示例，
特别是为SPARC平台创建静态编译器(也就是发射汇编码)，
因为SPARC具有相当典型的特征，
比如RISC指令集和常见的调用约定。

## 目标读者

本文档的受众是需要编写LLVM后端来为特定的硬件或软件平台生成代码的开发者。

## 前置知识

在阅读本文档之前，必须先阅读以下重要文档：

- [LLVM Language Reference Manual](https://releases.llvm.org/11.0.0/docs/LangRef.html)--LLVM汇编语言的参考手册。

- [The LLVM Target-Independent Code Generator](https://releases.llvm.org/11.0.0/docs/CodeGenerator.html)--
用于将LLVM内部表示转换为指定平台的机器码的组件（类和代码生成算法）的指南。
特别注意代码生成阶段：指令选择、调度和形成、基于SSA的优化、寄存器分配、Prolog/Epilog代码插入、后期机器代码优化和代码发射。

- [TableGen](https://releases.llvm.org/11.0.0/docs/TableGen/index.html)--TableGen（tblgen）应用程序的描述文档，
该程序管理LLVM代码生成所需的特定信息。
TableGen能把目标描述文件（后缀.td）转换成用于代码生成的C++代码。

## 基本步骤

编写将LLVM IR转换为指定平台的机器码或其他语言的代码的编译器后端，需要以下步骤：

- 创建TargetMachine类的子类，
该类描述目标计算机的特征，
这一步可以参考已有后端的TargetMachine类及其头文件；
例如，直接复制SparcTargetMachine.cpp和SparcTargetMachine.h但更改其文件名，
并且更改引用“Sparc”的代码以引用您的代码。
- 描述目标平台的寄存器。
使用TableGen从RegisterInfo.td文件生成定义寄存器、寄存器别名和寄存器组的代码。
还可能需要为TargetRegisterInfo的子类编写代码，这些代码代用于支持寄存器的分配以及描述寄存器间的约束。
- 描述目标平台的指令集。
使用TableGen从TargetInstrFormats.td和TargetInstrInfo.td文件生成描述目标平台的指令集的代码。
您可能需要手动为TargetInstrInfo的子类编写代码，以描述目标平台支持的某些特殊指令。
- 描述指令选择规则，该过程将LLVM IR的有向无环图（DAG）表示转换到目标平台原生指令表示。
使用TableGen根据TargetInstrInfo.td文件从定义的模式来生成支持指令选择的代码，
有时需要手动为XXXISelDAGToDAG.cpp编写代码来完成DAG-to-DAG的转换，
有时还需要手动为XXXISelLowering.cpp文件编写代码来替换不被SelectionDAG原生支持的操作和数据类型。
- 编写汇编生成器，汇编生成器将LLVM IR转换为目标计算机的GAS格式。
你需要在TargetInstrInfo.td文件中增加assembly strings。
同时还需要实现AsmPrinter的子类以及TargetAsmInfo的子类，
来实现LLVM IR到汇编的转换。
- （可选）添加对子平台(具有不同功能的变体)的支持。
实现TargetSubtarget的子类，
该类允许您使用`-mcpu=`和`-mattr=`命令行选项。
- （可选）添加JIT支持并创建机器码发射器(TargetJITInfo的子类)，
它用于直接将二进制代码发送到内存中。

在.cpp和.h文件中，首先为这些方法建立占位，然后在以后实现它们。
最初，您可能不知道这些类需要哪些私有成员，哪些类需要子类化。

## 准备工作

