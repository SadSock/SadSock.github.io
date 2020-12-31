---
layout: post
title:  "TableGen 简介"
toc: true
date:   2020-11-03 19:30:38 +0800
categories: LLVM
keywords: LLVM
description: 无
---


# 简介

TableGen的目的是帮助开发者开发和维护特定领域的记录。这些记录的数量可能很大，所以它是专门来编写灵活的描述，并提取这些记录的共同特征。这减少了描述的重复量，减少了出错的机会，使构建特定领域的信息更容易。

TableGen前端解析文件，实例化声明，并将结果交给特定领域的[backend](http://llvm.org/docs/TableGen/index.html#backend)进行处理。有关TableGen的详细描述，请参阅 [TableGen Programmer’s Reference](http://llvm.org/docs/TableGen/ProgRef.html) 。有关调用TableGen的各种xxx-tblgen命令的详细信息，请参阅[xxx-tblgen - Target Description to C++ Code](http://llvm.org/docs/CommandGuide/tblgen.html) 。

目前TableGen的主要用户有 [The LLVM Target-Independent Code Generator](http://llvm.org/docs/CodeGenerator.html)和[Clang diagnostics and attributes](https://clang.llvm.org/docs/UsersManual.html#controlling-errors-and-warnings)。

注意，如果您经常使用TableGen，并且使用emacs或vim，那么您可以在llvm发行版的`llvm/utils/emacs`和`llvm/utils/vim`目录中分别找到一个emacs“TableGen mode”和一个vim语言文件。

# TableGen程序

TableGen文件由TableGen程序解释：llvm-tblgen在bin下构建目录中。 它没有安装到系统（或您的sysroot设置的位置）中，因为它在LLVM的构建过程之外没有任何用处。

## 运行TableGen

TableGen的运行就像其他任何LLVM工具一样。 第一个（可选）参数指定要读取的文件。 如果未指定文件名，则llvm-tblgen从标准输入读取。

要使用TableGen就必须指定后端，这些后端可以在命令行中选择（输入“ `llvm-tblgen -help`”获取列表）。例如，要获取特定类型的子类的所有定义的列表(这对于构建这些记录的枚举很有用)，请使用`print-enum`选项：

```bash
$ llvm-tblgen X86.td -print-enums -class=Register
AH, AL, AX, BH, BL, BP, BPL, BX, CH, CL, CX, DH, DI, DIL, DL, DX, EAX, EBP, EBX,
ECX, EDI, EDX, EFLAGS, EIP, ESI, ESP, FP0, FP1, FP2, FP3, FP4, FP5, FP6, IP,
MM0, MM1, MM2, MM3, MM4, MM5, MM6, MM7, R10, R10B, R10D, R10W, R11, R11B, R11D,
R11W, R12, R12B, R12D, R12W, R13, R13B, R13D, R13W, R14, R14B, R14D, R14W, R15,
R15B, R15D, R15W, R8, R8B, R8D, R8W, R9, R9B, R9D, R9W, RAX, RBP, RBX, RCX, RDI,
RDX, RIP, RSI, RSP, SI, SIL, SP, SPL, ST0, ST1, ST2, ST3, ST4, ST5, ST6, ST7,
XMM0, XMM1, XMM10, XMM11, XMM12, XMM13, XMM14, XMM15, XMM2, XMM3, XMM4, XMM5,
XMM6, XMM7, XMM8, XMM9,

$ llvm-tblgen X86.td -print-enums -class=Instruction
ABS_F, ABS_Fp32, ABS_Fp64, ABS_Fp80, ADC32mi, ADC32mi8, ADC32mr, ADC32ri,
ADC32ri8, ADC32rm, ADC32rr, ADC64mi32, ADC64mi8, ADC64mr, ADC64ri32, ADC64ri8,
ADC64rm, ADC64rr, ADD16mi, ADD16mi8, ADD16mr, ADD16ri, ADD16ri8, ADD16rm,
ADD16rr, ADD32mi, ADD32mi8, ADD32mr, ADD32ri, ADD32ri8, ADD32rm, ADD32rr,
ADD64mi32, ADD64mi8, ADD64mr, ADD64ri32, ...
```

默认后端打印出所有记录。 还有一个通用后端，它将所有记录输出为JSON数据结构，并使用-dump-json选项启用。

如果您计划使用TableGen，那么您很可能必须编写一个后端来提取您需要的信息并以适当的方式格式化它。可以通过用C++扩展TableGen本身，或者用任何可以处理JSON的语言编写脚本来实现这一点。

## 例子

在没有其他参数的情况下，llvm-tblgen 解析指定的文件并输出所有的类，然后输出所有的定义。这是查看各种定义扩展到何种程度的好方法。在 X86.td 文件上运行这个命令会输出以下命令(在撰写本文时) :

```rust
...
def ADD32rr {   // Instruction X86Inst I
  string Namespace = "X86";
  dag OutOperandList = (outs GR32:$dst);
  dag InOperandList = (ins GR32:$src1, GR32:$src2);
  string AsmString = "add{l}\t{$src2, $dst|$dst, $src2}";
  list<dag> Pattern = [(set GR32:$dst, (add GR32:$src1, GR32:$src2))];
  list<Register> Uses = [];
  list<Register> Defs = [EFLAGS];
  list<Predicate> Predicates = [];
  int CodeSize = 3;
  int AddedComplexity = 0;
  bit isReturn = 0;
  bit isBranch = 0;
  bit isIndirectBranch = 0;
  bit isBarrier = 0;
  bit isCall = 0;
  bit canFoldAsLoad = 0;
  bit mayLoad = 0;
  bit mayStore = 0;
  bit isImplicitDef = 0;
  bit isConvertibleToThreeAddress = 1;
  bit isCommutable = 1;
  bit isTerminator = 0;
  bit isReMaterializable = 0;
  bit isPredicable = 0;
  bit hasDelaySlot = 0;
  bit usesCustomInserter = 0;
  bit hasCtrlDep = 0;
  bit isNotDuplicable = 0;
  bit hasSideEffects = 0;
  InstrItinClass Itinerary = NoItinerary;
  string Constraints = "";
  string DisableEncoding = "";
  bits<8> Opcode = { 0, 0, 0, 0, 0, 0, 0, 1 };
  Format Form = MRMDestReg;
  bits<6> FormBits = { 0, 0, 0, 0, 1, 1 };
  ImmType ImmT = NoImm;
  bits<3> ImmTypeBits = { 0, 0, 0 };
  bit hasOpSizePrefix = 0;
  bit hasAdSizePrefix = 0;
  bits<4> Prefix = { 0, 0, 0, 0 };
  bit hasREX_WPrefix = 0;
  FPFormat FPForm = ?;
  bits<3> FPFormBits = { 0, 0, 0 };
}
...
```

该定义对应于x86体系结构的32位寄存器-寄存器相加指令。 Def ADD32rr定义了一个名为ADD32rr的记录，行尾的注释表示该定义的父类。 记录体包含TableGen为记录汇编的所有数据，指示该指令是“x86”命名空间的一部分，指示代码生成器如何选择指令的模式，它是双地址指令，具有特定的编码等。记录中信息的内容和语义专用于X86后端，这里仅作为示例。

正如您所看到的，代码生成器支持的每条指令都需要大量信息，手动指定所有指令将不可维护，容易出错，而且首先要做的事情很累人。因为我们正在使用 TableGen，所有的信息都来自以下定义:

```rust
let Defs = [EFLAGS],
    isCommutable = 1,                  // X = ADD Y,Z --> X = ADD Z,Y
    isConvertibleToThreeAddress = 1 in // Can transform into LEA.
def ADD32rr  : I<0x01, MRMDestReg, (outs GR32:$dst),
                                   (ins GR32:$src1, GR32:$src2),
                 "add{l}\t{$src2, $dst|$dst, $src2}",
                 [(set GR32:$dst, (add GR32:$src1, GR32:$src2))]>;
```

这个定义使用定制类 I (从定制类 X86Inst 扩展而来) ，这个定制类是在 x86特定的 TableGen 文件中定义的，可以提取了指令共享的公共特性。TableGen 的一个关键特性是，它允许最终用户定义他们在描述其信息时喜欢使用的抽象。

# 语法

TableGen的语法类似c++模板，带有内置的类型和约束。此外，TableGen的语法引入了一些自动化概念，如multiclass、foreach、let等。

## 基本概念

TableGen文件由两个关键部分组成:“类”和“定义”，这两个部分都被认为是“记录”。

**记录** 有一个唯一的名称、一个值列表和一个父类列表。 值列表是TableGen为每条记录构建的主要数据；正是该列表保存了应用程序的特定信息。 此数据的解释留给特定的后端，但结构和格式规则由TableGen负责。

**定义** 是“记录”的具体形式。它通常不包含任何未定义的值，并用“ def”关键字进行标记。

```rust
def FeatureFPARMv8 : SubtargetFeature<"fp-armv8", "HasFPARMv8", "true",
                                      "Enable ARMv8 FP">;
```

在此示例中，FeatureFPARMv8是使用某些值初始化的SubtargetFeature记录。 这些类通过关键字class定义在同一文件或其他文件中的。 对大多数平台来说，包含通用类的TableGen文件保存在`include/llvm/Target`中。

**class** 是用于构建和描述其他记录的抽象记录。这些类使最终用户可以为他们所针对的领域（例如LLVM代码生成器中的“ Register”，“ RegisterClass”和“ Instruction”）构建抽象，提取改领域的公共属性（例如“ FPInst”，用于表示X86后端中的浮点指令）。TableGen会跟踪用于建立定义的所有类，因此后端可以找到特定类的所有定义，例如“instruction”。

```rust
class ProcNoItin<string Name, list<SubtargetFeature> Features>
      : Processor<Name, NoItineraries, Features>;
```

在这里，类ProcNoItin通过类型为string的参数名、目标特性列表和硬编码的NoItineraries来特化Processor类。

**multiclass** 一次实例化一组抽象记录。每个实例化都会产生多个TableGen定义。如果一个multiclass继承了另一个multiclass，那么子multiclass中的定义将成为当前multiclass的一部分，就像它们是在当前multiclass中声明的一样。

```rust
multiclass ro_signed_pats<string T, string Rm, dag Base, dag Offset, dag Extend,
                        dag address, ValueType sty> {
def : Pat<(i32 (!cast<SDNode>("sextload" # sty) address)),
          (!cast<Instruction>("LDRS" # T # "w_" # Rm # "_RegOffset")
            Base, Offset, Extend)>;

def : Pat<(i64 (!cast<SDNode>("sextload" # sty) address)),
          (!cast<Instruction>("LDRS" # T # "x_" # Rm # "_RegOffset")
            Base, Offset, Extend)>;
}

defm : ro_signed_pats<"B", Rm, Base, Offset, Extend,
                      !foreach(decls.pattern, address,
                               !subst(SHIFT, imm_eq0, decls.pattern)),
                      i8>;
```

有关TableGen的深入描述，请参阅TableGen的[TableGen Programmer’s Reference](http://llvm.org/docs/TableGen/ProgRef.html)。

# TableGen后端

没有后端，TableGen文件没有实际含义。 运行xxx-tblgen时的默认操作是以文本格式打印信息，但这仅对调试TableGen文件本身有用。 但是，TableGen的功能是将源文件解释为内部表示形式，可以将其生成为所需的任何内容。

目前TableGen的用法是创建包含表的大型include的文件，您可以直接包含这些文件(如果输出是您正在编码的语言) ，也可以通过包裹include文件的宏进行预处理。

如果后端已经以C格式打印了表，或者输出仅仅是一列字符串(用于错误和警告消息) ，则可以使用直接输出。如果需要在不同的上下文中使用相同的信息(如指令名) ，则应该使用预处理的输出，因此后端应该打印一个元信息列表，该列表可以形成不同的编译时格式。

可用后端的列表，请参阅[TableGen BackEnds](http://llvm.org/docs/TableGen/BackEnds.html)；有关如何编写和调试新后端的信息，请参阅[TableGen Backend Developer’s Guide](http://llvm.org/docs/TableGen/BackGuide.html)。

# TableGen缺陷

尽管TableGen非常通用，但它有一些已经被多次指出的缺陷。 共同点是，虽然TableGen允许您构建特定于领域的语言，但您创建的最终语言缺乏其他DSL的功能，这反过来又会显著增加TableGen文件的大小和复杂性。

同时，TableGen允许您通过定制的后端创建基本概念的任何含义，这会扭曲原始设计，使新手很难理解TableGen文件。

有些人赞成进一步扩展语义，但要确保后端遵守严格的规则。 其他人则建议我们应该改用功能更强大的DSL，这些DSL是为特定目的而设计的，甚至应该重用现有DSL。
