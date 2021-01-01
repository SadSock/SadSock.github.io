---
layout: post
title:  "编写LLVM后端"
date:   2020-11-01 18:38:38 +0800
toc: true
categories: LLVM
keywords: LLVM
description: 无
---

# 一 简介

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
该程序管理LLVM代码生成特定于域的信息。
TableGen处理来自目标描述文件（后缀.td）的输入，并生成可用于代码生成的C++代码。

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

要创建实际的编译器后端，
您需要创建和修改一些文件。 
这里只讨论必须的操作。
但是要实际使用LLVM的目标独立代码生成器，
您必须执行[LLVM Target-Independent Code Generator](https://releases.llvm.org/11.0.0/docs/CodeGenerator.html)中描述的步骤。


首先，您应该在`lib/Target`目录下创建一个子目录来保存与您的目标相关的所有文件。
如果目标名为“Dummy”，则创建`lib/target/Dummy`目录。

在这个新目录中，创建文件`CMakeLists.txt`。
最简单的方法是复制另一个目标的`CMakeLists.txt`文件并对其进行修改。
它至少应该指定`LLVM_TARGET_DEFINITIONS`变量。
这个库可以作为一个整体命名为`LLVMDummy`(参见MIPS后端)。
也可以将库拆分为`LLVMDummyAsmPrinter`和`LLVMDummyAsmPrinter`，
后者应该在`lib/Target/Dummy`的子目录中实现(参见PowerPC后端)。


注意，这两个命名方案都被硬编码到`llvm-config`中。
使用任何其他命名方案都会迷惑`llvm-config`，
并在链接`llc`时产生许多(看起来不相关的)链接器错误。


要使您的后端能执行实际的工作，
您需要实现`TargetMachine`的子类。
这个实现通常位于文件`lib/Target/DummyTargetMachine.cpp`中，
`lib/Target`目录中的其它文件也应该被正确实现。
要使用LLVM的目标独立代码生成器，
您应该像所有机器后端一样：
创建`LLVMTargetMachine`的子类。（要从头开始创建目标，请创建TargetMachine的子类。）

要让LLVM真正构建并链接你的后端，
你需要用`-DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=Dummy`命令运行cmake。
这将构建您的后端，而不需要将其添加到后端的列表中。

后端到达稳定版后，可以将其添加到位于主`CMakeLists.txt`文件的`LLVM_ALL_TARGETS`变量中。







# 二 目标机器

`LLVMTargetMachine`被设计成实现了目标无关代码生成器的目标的基类。
`LLVMTargetMachine`类特化为实现了各种虚拟方法的具体目标类。
`LLVMTargetMachine`在`include/llvm/target/TargetMachine.h`中定义为`TargetMachine`的子类。
`TargetMachine`类(TargetMachine.cpp)还负责处理许多命令行选项。

要为特定的目标创建`LLVMTargetMachine`的子类，
首先要复制现有的`TargetMachine`类的类文件和头文件。
您应该修改您创建的文件的文件名以反映能该目标。
例如，对于`SPARC`，将文件命名为`SparcTargetMachine.h`和`SparcTargetMachine.cpp`。

对于目标机器XXX，
`XXXTargetMachine`必须实现一系列用于获取后端组件的对象的方法。
这些方法被命名为`get * Info`，
这些方法可以获取指令集(`getInstrInfo`)、寄存器(`getRegisterInfo`)、堆栈布局(`getFrameInfo`)等信息。
`XXXTargetMachine`还必须实现`getDataLayout`方法，
以访问数据特征(如数据类型大小和对齐要求)对象。

例如，对于SPARC目标，
头文件`SparcTargetMachine.h`声明了`get*Info`和`getDataLayout`等方法的原型，
这些方法的返回值都是`SparcTargetMachine`类的成员变量。
```llvm
namespace llvm {

class Module;

class SparcTargetMachine : public LLVMTargetMachine {
  const DataLayout DataLayout;       // Calculates type size & alignment
  SparcSubtarget Subtarget;
  SparcInstrInfo InstrInfo;
  TargetFrameInfo FrameInfo;

protected:
  virtual const TargetAsmInfo *createTargetAsmInfo() const;

public:
  SparcTargetMachine(const Module &M, const std::string &FS);

  virtual const SparcInstrInfo *getInstrInfo() const {return &InstrInfo; }
  virtual const TargetFrameInfo *getFrameInfo() const {return &FrameInfo; }
  virtual const TargetSubtarget *getSubtargetImpl() const{return &Subtarget; }
  virtual const TargetRegisterInfo *getRegisterInfo() const {
    return &InstrInfo.getRegisterInfo();
  }
  virtual const DataLayout *getDataLayout() const { return &DataLayout; }
  static unsigned getModuleMatchQuality(const Module &M);

  // Pass Pipeline Configuration
  virtual bool addInstSelector(PassManagerBase &PM, bool Fast);
  virtual bool addPreEmitPass(PassManagerBase &PM, bool Fast);
};

} // end namespace llvm
```

+ getInstrInfo()

+ getRegisterInfo()

+ getFrameInfo()

+ getDataLayout()

+ getSubtargetImpl()

对于某些目标，还需要支持以下方法：

+ getTargetLowering()

+ getJITInfo()

有些体系结构（如gpu）不支持跳转到任意位置，
使用屏蔽执行来实现分支，
使用包裹循环体的特殊指令来实现循环。
为了避免CFG修改引入不可还原的控制流，
而这些控制流又不被硬件支持，
目标必须在初始化时调用`setRequiresStructuredCFG(true)`。

此外，`XXXTargetMachine`构造函数应该指定一个`TargetDescription`字符串，
该字符串确定目标机器的数据布局，
包括指针大小、对齐方式和端序等特征。
例如，`SparcTargetMachine`的构造函数包含以下内容:

```llvm
SparcTargetMachine::SparcTargetMachine(const Module &M, const std::string &FS)
  : DataLayout("E-p:32:32-f128:128:128"),
    Subtarget(M, FS), InstrInfo(Subtarget),
    FrameInfo(TargetFrameInfo::StackGrowsDown, 8, 0) {
}
```
连字符分隔TargetDescription字符串的各个部分。

+ 字符串中的大写'e'表示目标数据模型是big-endian，小写'e'表示little-endian。

+ "p:"后面跟着指针信息:大小、ABI对齐和首选对齐。如果"p:"后面只有两个数字，那么第一个值是指针大小，第二个值是ABI和首选对齐方式。

+ 然后是表示数字类型对齐的字母：“i”、“f”、“v”或“a”（对应于整数、浮点、向量或
聚合）。“i”、“v”或“a”后跟ABI对齐和首选
对齐。“f”后跟三个值：第一个值表示长双精度的大小，然后是ABI对齐，然后是ABI首选对齐。


# 三 目标注册
您还必须向`TargetRegistry`注册您的目标，
这是其他LLVM工具在运行时查找和使用您的目标的工具。
`TargetRegistry`可以直接使用，
但是对于大多数目标来说，
有一些辅助模板可以帮助您完成工作。


所有目标应该声明一个全局`Target`对象，
用于在注册期间表示目标。
然后，在目标的`TargetInfo`库中，
目标应该定义该对象并使用`RegisterTarget`模板注册目标。
例如，Sparc注册代码如下:
```llvm
Target llvm::getTheSparcTarget();

extern "C" void LLVMInitializeSparcTargetInfo() {
  RegisterTarget<Triple::sparc, /*HasJIT=*/false>
    X(getTheSparcTarget(), "sparc", "Sparc");
}
```
这允许`TargetRegistry`按名称或按目标三元组查找目标。
此外，大多数目标还将注册在单独的库中可用的其他特性。
这些注册步骤是分开的，因为有些客户可能希望只链接目标的某些部分--例如，
JIT代码生成器不需要使用汇编打印机。
下面是一个注册Sparc汇编输出器的例子:
```llvm
extern "C" void LLVMInitializeSparcAsmPrinter() {
  RegisterAsmPrinter<SparcAsmPrinter> X(getTheSparcTarget());
}
```
更多信息, 请参照“[llvm/Target/TargetRegistry.h](https://releases.llvm.org/doxygen/TargetRegistry_8h-source.html)”.

# 四 寄存器和寄存器组

（译注：本节及后文将原文中Register Set译为寄存器集合，将Register Class根据原文的含义译为寄存器组或Register类。）

您需要创建一个具体的寄存器描述类，这个类称为XXXRegisterInfo(其中XXX是平台标识符)，它描述了寄存器间的约束并为寄存器分配器提供必要的信息。

您还需要定义寄存器组来对相关寄存器进行分类。同一寄存器组的寄存器可以被某些指令以相同的方式使用。典型的例子是用于整数、浮点或向量的寄存器组。寄存器分配器允许指令以类似的方式使用同一寄存器组中的任何寄存器。寄存器分配器先给指令分配虚拟寄存器，然后会在寄存器分配阶段分配物理寄存器。

描述寄存器的大部分代码，包括寄存器定义、寄存器别名和寄存器组，都可以由TableGen工具自动生成。TableGen会根据开发者编写的xxxRegisterinfo.td文件，生成XXXGenRegisterInfo.h.inc和XXXGenRegisterInfo.inc文件。XXXRegisterInfo的实现过程中的一些代码需要手工编码。

## 4.1 定义寄存器

XXXRegisterinfo.td文件通常以目标机器的寄存器定义开始。Register类(在Target.td中定义)用于为每个寄存器定义一个对象。字符串n就是寄存器的名称。基本的Register对象不包含子寄存器，也没有指定别名。

```cpp
class Register<string n> {
  string Namespace = "";
  string AsmName = n;
  string Name = n;
  int SpillSize = 0;
  int SpillAlignment = 0;
  list<Register> Aliases = [];
  list<Register> SubRegs = [];
  list<int> DwarfNumbers = [];
}
```

例如，X86RegisterInfo.td文件使用Register类定义寄存器。比如：

```cpp
def AL : Register<"AL">, DwarfRegNum<[0, 0, 0]>;
```

这行代码定义了寄存器AL并使用DwarfRegNum为其赋值，gcc，gdb或其他调试信息工具用该值来识别寄存器。对于AL寄存器来说，DwarfRegNum使用了一个由3个值组成的数组，用来表示3 种不同的模式：第一个值是用于X86-64，第二个值用于X86-32中的异常处理（exception handling），第三个是通用值。-1表示gcc的值未定义，-2表示寄存器在该模式下是非法的。

根据X86RegisterInfo.td文件的描述，TableGen会在X86GenRegisterInfo.inc文件中生成以下代码：

```cpp
static const unsigned GR8[] = { X86::AL, ... };

const unsigned AL_AliasSet[] = { X86::AX, X86::EAX, X86::RAX, 0 };

const TargetRegisterDesc RegisterDescriptors[] = {
  ...
{ "AL", "AL", AL_AliasSet, Empty_SubRegsSet, Empty_SubRegsSet, AL_SuperRegsSet }, ...
```

根据register info文件，TableGen为每个寄存器生成一个TargetRegisterDesc对象。TargetRegisterDesc在`include/llvm/Target/Target.h`中被定义，包含以下字段：

```cpp
struct TargetRegisterDesc {
  const char     *AsmName;      // Assembly language name for the register
  const char     *Name;         // Printable name for the reg (for debugging)
  const unsigned *AliasSet;     // Register Alias Set
  const unsigned *SubRegs;      // Sub-register set
  const unsigned *ImmSubRegs;   // Immediate sub-register set
  const unsigned *SuperRegs;    // Super-register set
};
```

TableGen使用名称(TargetRegisterDesc的AsmName和Name字段)以及寄存器间的关系(TargetRegisterDesc的其他字段)来定义寄存器。在这个示例中，寄存器“AX”、“ EAX”和“RAX”为彼此的别名，TableGen为这个寄存器别名集生成一个以null结尾的数组(AL_aliasset)。

Register类通常用作更复杂类的基类。在Target.td中，Register类是RegisterWithSubRegs类的基类，该类用于定义需要在SubRegs列表中指定子寄存器的寄存器，如下所示

```cpp
class RegisterWithSubRegs<string n, list<Register> subregs> : Register<n> {
  let SubRegs = subregs;
}
```

SparcRegisterInfo.td为SPARC定义了额外的寄存器类：register类的子类SparcReg和其进一步的子类：Ri、Rf和Rd。SPARC的寄存器由5位ID号标识，这是这些子类的一个共同特性。“let”表达式可以覆盖最初在父类中定义的值（例如Rd类中的subgros字段）。

```cpp
class SparcReg<string n> : Register<n> {
  field bits<5> Num;
  let Namespace = "SP";
}
// Ri - 32-bit integer registers
class Ri<bits<5> num, string n> :
SparcReg<n> {
  let Num = num;
}
// Rf - 32-bit floating-point registers
class Rf<bits<5> num, string n> :
SparcReg<n> {
  let Num = num;
}
// Rd - Slots in the FP register file for 64-bit floating-point values.
class Rd<bits<5> num, string n, list<Register> subregs> : SparcReg<n> {
  let Num = num;
  let SubRegs = subregs;
}
```

SparcRegisterInfo.td文件利用Register类的子类来定义寄存器，例如

```python
def G0 : Ri< 0, "G0">, DwarfRegNum<[0]>;
def G1 : Ri< 1, "G1">, DwarfRegNum<[1]>;
...
def F0 : Rf< 0, "F0">, DwarfRegNum<[32]>;
def F1 : Rf< 1, "F1">, DwarfRegNum<[33]>;
...
def D0 : Rd< 0, "F0", [F0, F1]>, DwarfRegNum<[32]>;
def D1 : Rd< 2, "F2", [F2, F3]>, DwarfRegNum<[34]>;
```

上面显示的最后两个寄存器(D0和D1)是双精度浮点寄存器，它们是单精度浮点子寄存器对的别名。除了别名之外，子寄存器和父寄存器的关系也定义在TargetRegisterDesc的某些字段中。

## 4.2 定义寄存器组

`RegisterClass`类（在Target.td中指定）用于定义一个对象，
该对象表示一组相关的寄存器，
还定义了寄存器的默认分配顺序。
使用`Target.td`的目标描述文件`XXXRegisterInfo.td`可以使用以下类构造寄存器组：
```llvm
class RegisterClass<string namespace,
list<ValueType> regTypes, int alignment, dag regList> {
  string Namespace = namespace;
  list<ValueType> RegTypes = regTypes;
  int Size = 0;  // spill size, in bits; zero lets tblgen pick the size
  int Alignment = alignment;

  // CopyCost is the cost of copying a value between two registers
  // default value 1 means a single instruction
  // A negative value means copying is extremely expensive or impossible
  int CopyCost = 1;
  dag MemberList = regList;

  // for register classes that are subregisters of this class
  list<RegisterClass> SubRegClassList = [];

  code MethodProtos = [{}];  // to insert arbitrary code
  code MethodBodies = [{}];
}
```
要定义`RegisterClass`，请使用以下4个参数:

+ 第一个参数定义了命名空间的名称。

+ 第二个参数是寄存器类型的列表，寄存器的类型定义在文件`include/llvm/CodeGen/ValueTypes.td`中。
已定义的值包括整数类型(`i16`、`i32`和`i1`(布尔值))、浮点类型(`f32`、`f64`)和向量类型(例如，`v8i16`表示`8xi16`向量)。
`RegisterClass`中的所有寄存器必须具有相同的`ValueType`，
但有些寄存器可以不同的配置存储向量数据。
例如，一个能够处理128位向量的寄存器也能处理16个8位整数元素，8个16位整数，4个32位整数，等等。

+ 第三个参数指定寄存器数据在`load`或`save`时所需的对齐方式。

+ 最后一个参数`regList`指定这个集合包含的寄存器。
如果没有指定寄存器的分配顺序，
那么`regList`还暗含了寄存器的分配顺序。
除了简单地用`(add R0，R1，...)`列出寄存器之外，
还可以用更高级的集合操作符。
更多信息，请参见`include/llvm/Target/Target.td`。

在`SparcRegisterInfo.td`中，
定义了三个`RegisterClass`对象：`FPReg`，`DFPReg`和`IntReg`。
对于所有三个寄存器类，第一个参数都是使用字符串“ SP”定义名称空间。
`FPRegs`定义了一组32个单精度浮点寄存器（F0至F31）。
`DFPRegs`定义了一组16个双精度寄存器（D0-D15）。

```llvm
// F0, F1, F2, ..., F31
def FPRegs : RegisterClass<"SP", [f32], 32, (sequence "F%u", 0, 31)>;

def DFPRegs : RegisterClass<"SP", [f64], 64,
                            (add D0, D1, D2, D3, D4, D5, D6, D7, D8,
                                 D9, D10, D11, D12, D13, D14, D15)>;

def IntRegs : RegisterClass<"SP", [i32], 32,
    (add L0, L1, L2, L3, L4, L5, L6, L7,
         I0, I1, I2, I3, I4, I5,
         O0, O1, O2, O3, O4, O5, O7,
         G1,
         // Non-allocatable regs:
         G2, G3, G4,
         O6,        // stack ptr
         I6,        // frame ptr
         I7,        // return address
         G0,        // constant zero
         G5, G6, G7 // reserved for kernel
    )>;
```
TableGen将`SparcRegisterInfo.td`编译成多个输出文件，
这些输出文件将会被包含在您编写的其他源代码中。 
`SparcRegisterInfo.td`被编译成`SparcGenRegisterInfo.h.inc`，
这个文件将被包含在实现SPARC寄存器的头文件`SparcRegisterInfo.h`中。
`SparcGenRegisterInfo.h.inc`定义了一个名为`SparcGenRegisterInfo`的新结构，
该结构继承`TargetRegisterInfo`，
还根据预定义的寄存器集（`DFPRegsClass`，`FPRegsClass`和`IntRegsClass`）来指定类型。


`Sparcregisterinfo.td`还会生成`SparcGenRegisterInfo.inc`文件，
它被包含在文件`SparcRegisterInfo.cpp`的底部，
该文件用于实现Sparc的寄存器。
下面只显示生成的整数寄存器和关联的寄存器集，
`IntRegs`中寄存器的顺序同目标描述文件中`IntRegs`定义的顺序一致。
```cpp
// IntRegs Register Class...
static const unsigned IntRegs[] = {
  SP::L0, SP::L1, SP::L2, SP::L3, SP::L4, SP::L5,
  SP::L6, SP::L7, SP::I0, SP::I1, SP::I2, SP::I3,
  SP::I4, SP::I5, SP::O0, SP::O1, SP::O2, SP::O3,
  SP::O4, SP::O5, SP::O7, SP::G1, SP::G2, SP::G3,
  SP::G4, SP::O6, SP::I6, SP::I7, SP::G0, SP::G5,
  SP::G6, SP::G7,
};

// IntRegsVTs Register Class Value Types...
static const MVT::ValueType IntRegsVTs[] = {
  MVT::i32, MVT::Other
};

namespace SP {   // Register class instances
  DFPRegsClass    DFPRegsRegClass;
  FPRegsClass     FPRegsRegClass;
  IntRegsClass    IntRegsRegClass;
...
  // IntRegs Sub-register Classes...
  static const TargetRegisterClass* const IntRegsSubRegClasses [] = {
    NULL
  };
...
  // IntRegs Super-register Classes..
  static const TargetRegisterClass* const IntRegsSuperRegClasses [] = {
    NULL
  };
...
  // IntRegs Register Class sub-classes...
  static const TargetRegisterClass* const IntRegsSubclasses [] = {
    NULL
  };
...
  // IntRegs Register Class super-classes...
  static const TargetRegisterClass* const IntRegsSuperclasses [] = {
    NULL
  };

  IntRegsClass::IntRegsClass() : TargetRegisterClass(IntRegsRegClassID,
    IntRegsVTs, IntRegsSubclasses, IntRegsSuperclasses, IntRegsSubRegClasses,
    IntRegsSuperRegClasses, 4, 4, 1, IntRegs, IntRegs + 32) {}
}
```
寄存器分配器将避免使用保留寄存器，
并且被调用方保存的寄存器在所有易失性寄存器被使用之前都不会被使用。
这通常已经足够好了，
但在某些情况下，
可能需要提供自定义分配命令。

## 4.3 实现TargetRegisterInfo的子类

最后一步是手工编写`XXXRegisterInfo`的部分代码，
它实现了文件`TargetRegisterInfo.h`描述的接口（请参见`TargetRegisterInfo`类）。
如果不实现这些接口，这些接口将返回`0`、`NULL`或`false`。
下面是为实现SPARC而在文件`SparcRegisterInfo.cpp`中手工编写函数列表:

* `getCalleeSavedRegs` —- 返回被叫方保存的寄存器列表，按被叫方所需的堆栈帧偏移量顺序。

* `getReservedRegs` —- 返回物理寄存器索引的集合，指示特定寄存器是否不可用。

* `hasFP` —- 返回一个布尔值，指示函数是否应具有专用的帧指针寄存器。

* `eliminateCallFramePseudoInstr` —- 如果使用调用帧设置或销毁伪指令，则可以调用此命令来消除它们。

* `excludeFrameIndex` -- 从能使用抽象帧索引的指令中删除抽象帧索引。

* `emitPrologue` -- 在函数中插入Prologue代码。

* `emitEpilogue` -- 在函数中插入Epilogue代码。



# 五 指令集

在代码生成的早期阶段，
LLVM IR代码被转换为Selection DAG，
节点是`SDNode`类的实例，
`SDNode`类包含目标指令，
具有操作码、操作数、类型要求和操作属性。
例如，操作是否是可交换的，是否需要从内存加载数据。
文件`include/llvm/CodeGen/SelectionDAGNodes.h`（ISD命名空间中的NodeType枚举）描述了节点的各种类型。

## 指令操作数映射

### 指令操作数名称映射

### 指令操作数类型

## 指令调度

## 指令关系映射

## 实现TargetStrInfo的子类

## 分支折叠与If转换

# 六 指令选择器

## 选择合法化阶段

### 推广

### 展开

### 定制

### 合法的

## 调用约定


# 七 装配式打印机

# 八 子目标支持

# 九 JIT支持

## 机器码发射器

## 目标JIT信息
