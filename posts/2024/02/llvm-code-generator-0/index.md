# LLVM目标无关代码生成器（上）（译）


**提示**：*本文是LLVM官方文档[The LLVM Target-Independent Code Generator](https://llvm.org/docs/CodeGenerator.html)的译文上半部分*

## 简介

LLVM的代码生成器是一个框架，提供了一系列可重用组件，将LLVM IR翻译为特定目标机器的汇编指令。包含以下六个组件：

1. 抽象目标机描述接口：关于目标机器的重要属性（不包括这些属性是如何使用的）。位于include/llvm/Target。

2. 用于表示目标机器代码的类：使之足够抽象以便能表达任意目标机的机器代码。位于include/llvm/CodeGen。例如，可以表达"常量池条目"和"跳转表"的概念。

3. 用于在目标文件（MC层）表示代码的类和算法：可以表达汇编级的概念如：label, section, instruction. 而"常量池条目"和"跳转表"的概念不在这层表达。

4. 目标无关算法。用于实现不同阶段的本地代码生成，如：寄存器分配、指令调度、栈帧表示。位于lib/CodeGen

5. 对应于抽象机器描述接口的特定机器实现。这些机器描述利用LLVM提供的组件，以及为特定目标机定制的pass，为特定机器构件一个完整的目标生成器。位于lib/Target。

6. 目标无关的JIT组件。该组件本身是目标无关的，但它会利用TargetJITInfo接口来处理目标相关的问题。位于lib/ExecutionEngine/JIT。

后端开发人员需要熟悉目标机描述以及机器代码表示相关的类。若要为新机器添加后端，需要为该机器实现抽象机器描述接口，并理解LLVM IR。若要实现新的代码生成算法，为了其可移植性，必须只依靠来目标机描述和机器代码表示来做。

### 代码生成器中的必需组件

要在LLVM框架中增加后端，需要实现的必不可少的接口只有两个：TargetMachine, DataLayout。

这意味着：一方面LLVM能够支持非传统的目标机，比如以C语言为后端，就不需要寄存器分配、指令选择等阶段；另一方面可以设计实现出完全不同的不需要用LLVM内置组件的代码生成器，比如针对FPGA。

### 代码生成器的高层设计

设计代码生成器是为了支持寄存器微处理器的高效高质量代码生成。代码生成包含以下阶段：

1. 指令选择：将LLVM IR转换为目标指令DAG。该DAG使用SSA形式的虚拟寄存器，以及与目标约束或调用约定相关的特定物理寄存器。

2. 调度和生成序列化指令：以SelectionDAG为输入，决定指令顺序，并输出MachineInstr机器指令。

3. 基于SSA的机器代码优化：比如modulo-scheduling和窥孔优化。

4. 寄存器分配：将无限的虚拟寄存器映射到具体的物理寄存器，在必要时生成spill code。

5. 插入prolog/epilog：一旦机器码生成完毕，栈空间需求量确定，就可以插入prolog/epilog。帧指针消除和stack packing也是在这阶段做的。

6. 晚期机器码优化：比如spill code scheduling和窥孔优化。

7. 代码输出：可以是汇编格式或二进制格式。

指令选择是基于最优化的模式匹配选择器做的。

具体后端的实现可以在这些阶段中任意插入自己特定的Pass。

### 用TableGen描述目标机器

机器描述类需要关于目标架构的详细描述。为了提取出关于机器描述的公共部分，LLVM使用TableGen语言来描述目标机器，来减少重复。

随着LLVM继续发展，会将越来越多机器描述移到td文件中。这样可以使移植后端更容易。

## 目标机描述类（include/llvm/Target）

提供了目标机器的抽象描述。例如：指令、寄存器

所有的机器描述类（除了DataLayout）都是要被具体机器的实现继承的。TargetMachine类提供了需要被目标机实现的访问器。

### TargetMachine类

提供了用于访问不同机器描述类的虚函数（getInstrInfo, getRegisterInfo, getFrameInfo）。具体机器（X86TargetMachine）实现这些虚函数。

### DataLayout类

该类是唯一必需的机器描述类，且不可被继承。该类用来说明目标机针对不同数据类型的内存布局、对齐要求、指针size、大小端等信息。

### TargetLowering类

给指令选择使用，主要用于描述LLVM IR代码是如何lower到SelectionDAG。此外还用来指定：

1. 针对不同数据类型的register class
2. 目标机器支持的操作
3. setcc操作的返回类型
4. 用于表示Shfit amount的类型
5. 一些高层特性，如将常量除法替换为乘法序列是否有收益

### TargetRegisterInfo类

该类描述目标机的寄存器集合以及这些寄存器之间的相互作用。

寄存器是用无符号整数表示的。需注意寄存器0保留为flag值。

处理器描述中的每个寄存器都有一个关联的TargetRegisterDesc条目，该条目提供了寄存器的文本名字以及一系列别名。

TargetRegisterInfo类还暴露了处理器特定的寄存器类，每个寄存器类都有相同属性。指令选择器创建的每个SSA虚拟寄存器都有一个关联的寄存器类。寄存器分配时，虚拟寄存器被相关联的寄存器类中的一个物理寄存器所替代。这些寄存器类都是通过TableGen自动生成的。

### TargetInstrInfo类

该类描述目标机支持的指令。定义了opcode对应的助记符、操作数数量、隐式使用和定义的寄存器、是否具有一些机器无关的属性（访问内存、满足交换律等）和机器相关的flag。

### TargetFrameLowering类

提供关于目标机器堆栈布局的信息。包含堆栈的增长方向，每个函数的堆栈对齐要求，局部变量区域的偏移位置。

### TargetSubtarget类

提供目标机器的特定子芯片的信息。包括该子芯片支持的指令、指令延迟、指令指令itinerary（使用的的处理单元、顺序以及时长）。

### TargetJITInfo类

定义了给JIT代码生成器的抽象接口，来处理机器相关的活动，如发射stub。

## 机器码描述类

llvm使用MachineFunction, MachineBasicBlock, MachineInstr (include/llvm/CodeGen)
几个类来表示机器码，这几个类本身是机器无关的，将代码表示成抽象形式的指令：一个操作码加几个操作数。这种表示既支持SSA形式，也支持寄存器分配后的非SSA形式。

### MachineInstr类

将机器指令表示为MachineInstr的实例。MachineInstr表达指令的方式相当抽象，只记录操作码和一系列操作数。

操作码就是一个简单的无符号整数。目标机的所有指令都应该定义在InstrInfo.td中。操作码的枚举值就是从这些定义中生成的。MachineInstr类并不包含对应指令的语义信息（这些信息在TargetInstrInfo类中）。

操作数可以有几种类型：寄存器引用、常量整数、基本块引用等。另外，还需要将操作数标记为def或use。根据约定，操作数中寄存器def必须出现在use之前（不管最终的汇编码是否如此）。

#### 使用MachineInstrBuilder.h中的函数

可以使用BuildMI (include/llvm/CodeGen/MachineInstrBuilder.h)来创建机器指令。用法如下：
```c++
// Create a 'DestReg = mov 42' (rendered in X86 assembly as 'mov DestReg, 42')
// instruction and insert it at the end of the given MachineBasicBlock.
const TargetInstrInfo &TII = ...
MachineBasicBlock &MBB = ...
DebugLoc DL;
MachineInstr *MI = BuildMI(MBB, DL, TII.get(X86::MOV32ri), DestReg).addImm(42);

// Create the same instr, but insert it before a specified iterator point.
MachineBasicBlock::iterator MBBI = ...
BuildMI(MBB, MBBI, DL, TII.get(X86::MOV32ri), DestReg).addImm(42);

// Create a 'cmp Reg, 0' instruction, no destination reg.
MI = BuildMI(MBB, DL, TII.get(X86::CMP32ri8)).addReg(Reg).addImm(42);

// Create an 'sahf' instruction which takes no operands and stores nothing.
MI = BuildMI(MBB, DL, TII.get(X86::SAHF));

// Create a self looping branch instruction.
BuildMI(MBB, DL, TII.get(X86::JNE)).addMBB(&MBB);
```

若要增加额外的def操作数，必须要显式标记出来。
```c++
MI.addReg(Reg, RegState::Define);
```

#### 固定用途寄存器

代码生成器需要知道哪些寄存器是有固定用途的。指令流里面常常需要把特殊值安排在特殊寄存器中，比如由于指令集的限制，X86需要用EAX/EDX来做32位除法，再比如调用约定产生的特殊约束。在这些情况下，指令选择器会生成代码将虚拟寄存器拷入或拷出到物理寄存器。比如以下LLVM代码：
```llvm
define i32 @test(i32 %X, i32 %Y) {
  %Z = sdiv i32 %X, %Y
  ret i32 %Z
}
```

会被转换为如下的机器码：
```
;; Start of div
%EAX = mov %reg1024           ;; Copy X (in reg1024) into EAX
%reg1027 = sar %reg1024, 31
%EDX = mov %reg1027           ;; Sign extend X into EDX
idiv %reg1025                 ;; Divide by Y (in reg1025)
%reg1026 = mov %EAX           ;; Read the result (Z) out of EAX

;; Start of ret
%EAX = mov %reg1026           ;; 32-bit return value goes in EAX
ret
```

不过在最后，寄存器分配器会将冗余的寄存器拷贝操作删除，生成如下代码：
```
;; X is in EAX, Y is in ECX
mov %EAX, %EDX
sar %EDX, 31
idiv %ECX
ret
```

这种方法对任何目标机都适用。需要注意的是，为了高质量的代码生成，物理寄存器的生命周期要短，而且所有的物理寄存器在进入基本块和离开基本块时是dead的（在寄存器分配之前如此）。因此，如果需要一个值的活跃期跨越基本块，必须活跃于虚拟寄存器中。

#### 函数调用会破坏的寄存器

一些机器指令会破坏大量的物理寄存器，，比如函数调用。这时就需要用MO_RegisterMask来将它们标记出来，MO_RegisterMask能够标记出哪些寄存器是在函数调用中被保护的，而除此之外的其他寄存器就默认是被破坏了。

#### SSA形式的机器码

MachineInstr在寄存器分配之前都保持着SSA形式。LLVM PHI结点变成了机器码PHI结点，而虚拟寄存器只允许有一个def。在寄存器分配之后，代码中就不存在虚拟寄存器了，机器码也不再是SSA形式了。

### MachineBasicBlock类

包含一个机器指令(MachineInstr)列表。大致跟LLVM IR中的基本块是对应的，不过也有可能一个LLVM
IR基本块对应多个MachineBasicBlock。其中有一个getBasicBLock方法，可以返回对应的LLVM IR基本块。

### MachineFunction类

包含一个MachineBasicBlock列表。跟LLVM IR中的函数是对应的。除了MachineBasicBlock列表之外，每个MachineFunction还包含一个MachineConstantPool, 一个MachineFrameInfo, 一个MachineFunctionInfo, 一个MachineRegisterInfo。

### MachineInstr指令包

可以将一个指令序列视为MachineInstr指令包。这种指令包可以建模包含任意数量并行指令的VLIW指令，也可以建模无法分割的一个执行序列，如ARM Thumb2 IT block。

在概念上，一个MI指令包就是一个MI内部嵌套了几个其他MI。
```
--------------
|   Bundle   | ---------
--------------          \
       |           ----------------
       |           |      MI      |
       |           ----------------
       |                   |
       |           ----------------
       |           |      MI      |
       |           ----------------
       |                   |
       |           ----------------
       |           |      MI      |
       |           ----------------
       |
--------------
|   Bundle   | --------
--------------         \
       |           ----------------
       |           |      MI      |
       |           ----------------
       |                   |
       |           ----------------
       |           |      MI      |
       |           ----------------
       |                   |
       |                  ...
       |
--------------
|   Bundle   | --------
--------------         \
       |
      ...
```

内部的MI会打上InsideBundle标志。具有特殊BUNDLE操作码的顶层MI用来表示指令包的开头。

关于MachineInstr的Pass是将MI指令包作为基本单元来操作的。MachineBasicBlock的迭代器就是如此。正因如此，MachineBasicBlock还提供了另外一个迭代器instr_iterator，可以遍历基本块的所有MI指令。顶层BUNDLE指令必须具有正确的寄存器 MachineOperand 集合,这些寄存器表示捆绑指令(bundled MIs)的累积输入和输出。

对于VLIW架构，MachineInstr的打包是寄存器分配的一部分。具体说，决定哪些指令打包在一起的pass应该在代码从SSA形式退出之后做，需要在虚拟寄存器改写为物理寄存器之后完成。这就不再需要将虚拟寄存器操作数添加到BUNDLE指令中。

## MC层

MC层用来表示和处理原始机器码级别的代码，已经不再存在诸如常量池、跳转表、全局变量等高层次的概念。在MC层，LLVM处理诸如label名称，机器指令，目标文件中section等。这层的代码有几个重要目的：代码生成器用它生成.s或.o文件，llvm-mc工具用它实现独立的机器码汇编器和反汇编器。

以下讲述几个重要的相关类：

### MCStreamer API

最好将MCStreamer看做是汇编器API，它是抽象的，可以用不同方式实现，比如生成.s、生成.o就是不同的方式。MCStreamer对于每种directive都有一个方法，比如EmitLable, EmitSymbolAttribute, switchSection,
emitValue等，这些都对应着汇编级别的概念。还有一个EmitInstruction方法，用来输出MCInst。

这套API有两个重要的客户：llvm-mc独立汇编器和代码生成器的代码发射阶段。

MCStreamer有两个主要的实现：一个用来输出.s文件，另一个用来输出.o文件。输出.o文件的MCObjectStreamer实现了一个完整的汇编器。

对于目标机器特定的directive，MCStreamer有一个MCTargetStreamer实例，一个目标机如果需要实现特定的directive，那么就需要从该类继承。这个自定义的子类需要对每个directive定义一个方法，同时还需要定义两个子类，分别生成asm和obj。

目标机的初始化代码还需要调用TargetRegistry::RegisterAsmStreamer和TargetRegistry::RegisterMCObjectStreamer将对应的streamer进行注册。

### MCContext类

该类是MC层各种数据结构的所有者，包括symbols, sections等。若要创建symbol和section，就要与该类交互。该类不可被继承。

### MCSymbol类

该类代表汇编文件中的一个symbol (aka label)。symbol有两种：汇编器临时symbol、正常symbol。汇编器临时symbol是被汇编器使用并处理的，并在生成obj文件时被丢弃。常见的做法是在label名前加前缀以示区别。

MCSymbol是被MCContext创建的，并保证唯一性。这意味着两个MCSymbol可以通过比较指针是否相等来判断同一性。但是指针不相等却不能保证它们最终映射到不同的地址。如下的输出在.s文件中是合法的：
```
foo:
bar:
  .byte 4
```

这种情况下，foo和bar具有相同的地址。

### MCSection类

该类代表一个obj文件的section。可以被继承以实现各种不同格式的obj文件，如ELF。它是被MCContext创建的并保证唯一性。MCStreamer中有当前section的概念，可以通过SwitchToSection来改变当前section（这个概念与.s文件中的.section directive相当）。

### MCInst类

该类是一条指令的机器无关表示。这是一个很简单的类（相对于MachineInstr来说），只包含了特定机器的操作码和一系列MCOperands。而MCOperand只是一个包含了三种情况的union：1) 简单立即数 2）寄存器ID 3) 符号表达式MCExpr（如Lfoo-Lbar+42）

### 目标文件格式

MC层的obj文件输出器支持多种obj格式。实际上每个目标机支持的格式仅仅是MC层提供的子集。大部分机器支持ELF格式，而其他机器可能还有它们自己定义的格式。




---

> 作者: [lazypanda](https://github.com/wanghuibin0)  
> URL: https://simplecoding.fun/posts/2024/02/llvm-code-generator-0/  

