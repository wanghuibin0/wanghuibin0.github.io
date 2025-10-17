# LLVM目标无关代码生成器（下）（译）


**提示** ： *本文是LLVM官方文档[The LLVM Target-Independent Code Generator](https://llvm.org/docs/CodeGenerator.html)的译文下半部分*

## 机器无关代码生成算法

本节描述代码生成器的各个阶段，描述它们是如何工作的，以及如此设计的背后原理。

### 指令选择

指令选择是将LLVM IR转换为特定机器指令的过程。学术界有多种指令选择算法，而LLVM使用了基于SelectionDAG的指令选择器。

目前，DAG指令选择器的一部分是由.td文件生成的，另一部分还是需要用C++代码实现。

GlobalISel是另一个指令选择框架，目前还是实验版本。

#### SelectionDAG简介

SelectionDAG提供了一种代码表示方式，对于用自动化的指令选择比较友好。另外，对于指令调度也比较友好。而且在这种表示上还可以做一些很低级的机器无关优化。

SelectionDAG是一种有向无环图，其结点是SDNode实例。SDNode主要包含了操作码和操作数。include/llvm/ISDOpcodes.h文件中描述各种操作节点类型。

实际中大多结点只是定义一个值，但每个结点都有可能定义多个值。比如，一个div/rem操作同时定义了商和余数。很多其他情形也需要多个值。每个结点还有若干操作数，表示为指向定义这些操作数的结点的边。因为结点可能定义多个值，这些边要表示为SDValue实例，即<SDNode, unsigned>序对，SDNode表示来自哪个结点，unsigned表示来自其哪个结果。SDNode产生的每个值都有关联的MVT (Machine Value Type)，表示该值的类型。

SelectionDAG包含两种类型的值：表示数据依赖的，表示控制依赖的。数据值就是整数或浮点数类型的边，控制依赖边就是MVT::Other类型的chain边。这些边为有副作用的结点(load/store/call/return)提供了顺序。根据约定，chain输入总是0号操作数，chain输出总是该操作产生的最后一个值。然而，经过指令选择，机器结点也会有在指令操作数后面的chain输入，而且后面可能跟着glue结点。

SelectionDAG指定有Entry和Root结点，Entry结点是一个标记结点，其操作数总是ISD::EntryToken。Root结点是产生最终结果的结点。

对于SelectionDAG，一个重要的概念是合法或非法。合法DAG只使用目标机支持的操作和类型。类型合法化和操作合法化负责将非法DAG转换为合法DAG。

#### SelectionDAG指令选择过程

包含以下步骤：

1. 构建初始DAG：从LLVM IR到非法DAG做一个简单的转换。
2. 优化SelectionDAG：在DAG上做一些简单优化，并识别出目标机上支持的元操作，从而使得后面的转换更简单些。
3. 类型合法化：清除目标机器上不支持的类型。
4. 优化SelectionDAG：消除类型合法化引入的冗余。
5. 操作合法化：清除目标机器上不支持的操作。
6. 优化SelectionDAG：消除操作合法化引入的冗余。
7. 从DAG上做指令选择：将机器无关的DAG输入转换为目标机指令的DAG。
8. SelectionDAG调度和指令序列化：给DAG中的指令指定一个线性顺序，并输出为MachineFunction。这一步使用了传统的调度技术。

上述步骤完毕后，SelectionDAG被销毁，再运行的其他的代码生成pass。

为方便查看这些步骤发生的事情，可以利用LLC工具的一些选项。

- -view-dag-combine1-dags 显示初构建好的DAG。
- -view-legalize-dags 显示合法化前的DAG。
- -view-dag-combine2-dags 显示在第二步优化前的DAG。
- -view-isel-dags 显示在指令选择前的DAG。
- -view-sched-dags 显示在调度前的DAG。
- -view-sunit-dags 显示调度器的依赖图，该图基于最终SelectionDAG。
- -filter-view-dags 可以选择要查看的基本块的名称，并配合上面几个选项一起使用。

#### 初始DAG构建

SelectionDAGBuilder类负责从LLVM IR构建SelectionDAG，这个Pass大部分都是硬编码的（LLVM add -> SDNode add, getelementptr -> 显式的算术运算），过程中需要用到机器特定hooks来lower calls, returns, varargs等。

#### SelectionDAG类型合法化

使DAG只使用目标机支持的类型。

将不支持的标量类型转化为支持的类型有两种方式：一种叫promoting，将小类型转为大类型，另一种叫expanding，将大整型分解为几个小整型。

将不支持的vector类型转换为支持的类型也有两种方式：一种是将vector分割，直到找到合法的类型，另一种是通过在末尾添加元素的方式扩展vector，叫expanding。如果一个vector一路到底分割为了单个元素，还没有找到支持的vector类型，那么这些元素就会被转化为标量，叫scalarizing。

目标机实现通过调用TargetLowering中的addRegisterClass方法来告知合法化器它支持哪些类型，对应的寄存器类是什么。

#### SelectionDAG操作合法化

使DAG只使用目标机支持的操作。

目标机通常有古怪的约束，比如不支持某些操作。操作合法化通过用一系列操作来模拟某个不支持的操作（expansion）,或者通过将类型提升为更大类型来支持该操作（promotion）,或者通过目标特定的hook来实现合法化（custom）。

目标机实现通过调用TargetLowering中的setOperationAction方法来告知合法化器它不支持哪些操作，还有采用以上哪种方式来处理这种情况。

如果目标机有合法的vector类型，肯定希望用这些类型为shufflevector IR指令生成高效机器码，这就需要为shufflevector定制合法化操作。需要处理的形式包括以下几种：

- vector select: 就两个输入vector中挑选元素组成新的vector，这也叫做blend或者bitwise select。

- Insert subvector：vector要放入从index 0开始的一个更长的vector中。

- Extract subvector: 从一个长的vector的index 0处拉出子向量。

- Splat: vector的所有元素都是同样的标量元素。也叫broadcast或者duplicate。

合法化阶段的引入提供了一种SelectionDAG的标准化形式，并且可以在这种形式上做一些通用优化。

#### SelectionDAG优化阶段：DAG合并器

在代码生成中，SelectionDAG优化会多次运行，包括DAG刚刚构建好，以及每次合法化pass之后。该pass主要是做一些清理工作。一项重要的工作就是优化插入的符号扩展和零扩展指令。目前是使用ad-hoc技术，未来可能会转为使用更严格的技术。

#### SelectionDAG指令选择阶段

这是指令选择过程的与机器有关的主要阶段。该阶段以合法的SelectionDAG为输入，通过模式匹配选择目标机上支持的指令，并输出新的DAG。考虑如下LLVM IR:
```llvm
%t1 = fadd float %W, %X
%t2 = fmul float %t1, %Y
%t3 = fadd float %t2, %Z
```

对应的SelectionDAG大致如下：
```
(fadd:f32 (fmul:f32 (fadd:f32 W, X), Y), Z)
```

如果目标机支持浮点乘累加操作，那么乘和加就可以合并。在PowerPC上，指令选择器的输出大致如下：
```
(FMADDS (FADDS W, X), Y, Z)
```

PowerPC的后端会包含以下指令定义：
```
def FMADDS : AForm_1<59, 29,
                    (ops F4RC:$FRT, F4RC:$FRA, F4RC:$FRC, F4RC:$FRB),
                    "fmadds $FRT, $FRA, $FRC, $FRB",
                    [(set F4RC:$FRT, (fadd (fmul F4RC:$FRA, F4RC:$FRC),
                                           F4RC:$FRB))]>;
def FADDS : AForm_2<59, 21,
                    (ops F4RC:$FRT, F4RC:$FRA, F4RC:$FRB),
                    "fadds $FRT, $FRA, $FRB",
                    [(set F4RC:$FRT, (fadd F4RC:$FRA, F4RC:$FRB))]>;
```

TableGen DAG指令选择器从.td文件中读取指令模式，并自动构建模式匹配代码。它具有以下优势：

- 在编译器自身的编译阶段，可以分析指令模式并告诉你该模式是否有意义。

- 在模式匹配时处理操作数约束。比如，很方便表达"匹配一个13
bit的符号扩展立即数"。

- 知悉关于模式一些重要性质。比如，知悉加法符合交换律。

- 具有功能齐全的类型推理系统。基本上不需要显式说明模式的每个部分是什么类型的。

- 目标机可以定义它们自己的模式片段。模式片段就是命名并可重用的模式。

- 除了指令，目标机还可以利用Pat类指定可以映射到单个或多个指令模式。比如，PowerPC不具有在一条指令中奖任意整型立即数搬到寄存器的方式，那么就可以在tblgen中做如下定义：
```
// Arbitrary immediate support.  Implement in terms of LIS/ORI.
def : Pat<(i32 imm:$imm),
          (ORI (LIS (HI16 imm:$imm)), (LO16 imm:$imm))>;
```

- 当使用Pat类将一个模式映射为一条具有单个或多个操作数的指令时，该模式要么用ComplexPattern指定为一个整体，要么分开指定操作数的各个部分。比如，使用后者的PowerPC的后端做了如下定义：
```
def STWU  : DForm_1<37, (outs ptr_rc:$ea_res), (ins GPRC:$rS, memri:$dst),
                "stwu $rS, $dst", LdStStoreUpd, []>,
                RegConstraint<"$dst.reg = $ea_res">, NoEncode<"$ea_res">;

def : Pat<(pre_store GPRC:$rS, ptr_rc:$ptrreg, iaddroff:$ptroff),
          (STWU GPRC:$rS, iaddroff:$ptroff, ptr_rc:$ptrreg)>;
```

- 即便该系统已经很自动化了，仍然需要定制一些C++代码来匹配特殊情形。

尽管有很多优点，该系统目前也有一些缺陷，主要是由于还有一些未完成的工作：

- 没有提供能定义多个值的SelectionDAG结点。这是仍然需要定制C++代码的主要原因。

- 尚未有支持复杂寻址模式的好的方式。未来将会扩展模式片段使之能够定义多个值。

- 无法自动推导类似isStore/isLoad的flag。

- 无法为合法化器自动生成所支持的寄存器和操作集合。

- 还没有绑定自定义合法化结点的方法。

尽管有这些限制，指令选择器生成器对于大多数的二元操作和逻辑操作指令仍然非常有用。

#### SelectionDAG调度和指令序列化阶段

依据目标机器的各种约束，将机器指令的DAG指派一个顺序。顺序一旦建立，DAG就转换为MachineInstr列表，而后SelectionDAG就销毁了。

该阶段在逻辑上与指令选择是分开的，但是联系很紧密，因为该阶段的输入正是SelectionDAG。

### 基于SSA的机器码优化

TODO

### 活跃区间

活跃区间表示一个变量在哪个范围内是活跃的（live）。某些寄存器分配器会根据该信息确定需要相同物理寄存器的几个虚拟寄存器是否在某些程序点上同时活跃，即它们产生了冲突。如果产生了冲突，就要把某个虚拟寄存器溢出（spill）。

#### 活跃变量分析

确定变量活跃区间的第一步就是计算指令执行后立即失效的寄存器集合（即指令计算了值，但后续不再使用），和本条指令中使用但后续不再使用的寄存器集合（即被杀死）。需要对每个虚拟寄存器和可分配的物理寄存器计算活跃变量信息，由于可以使用SSA对虚拟寄存器进行稀疏生命期分析，而且只需要跟踪一个基本块内的物理寄存器，这项任务可以很高效地完成。在寄存器分配之前，LLVM假设物理寄存器只在单个基本块活跃，这就使得通过局部分析就可以解析出物理寄存器的生命期。而如果一个物理寄存器不是可分配的（比如栈指针或条件码），就不需要跟踪它。

物理寄存器可能以活跃的状态流入或流出一个函数。流入的值常常是寄存器传递的参数，流出的值常常是寄存器传递的返回值。这些值在活跃区间分析中要打上特殊的标记。

PHI结点需要特殊处理，主要是因为在计算活跃变量信息时，对CFG的DFS遍历无法保证PHI结点使用的虚拟寄存器在使用前定义。当遇到PHI结点时，只处理定义，这是因为其使用会在其他基本块中处理。

对于当前基本块的每个PHI结点，要在当前基本块末尾模拟一个赋值语句并遍历后继基本块。如果后继基本块有PHI结点，且PHI结点的操作数来自于当前基本块，那么该变量就标记为在当前基本块中以及所有的前驱基本块中都是活跃的，直到碰到定义该变量的那个基本块。

#### 活跃区间分析

现在有了足够的信息来做活跃区间分析。从给基本块和机器指令编号开始，接着处理流入的活跃值，这些值都是在物理寄存器中，所以在基本块末尾可以认为这些寄存器都死亡了。而对于虚拟寄存器，会计算出一个\[i,
j)形式的活跃区间，其中i, j都是指令编号。

### 寄存器分配

寄存器分配问题就是将一个使用无限虚拟寄存器的程序，映射为只使用有限物理寄存器的程序。每个机器架构都有不同数量的物理寄存器，如果物理寄存器不够容纳所有的虚拟寄存器，一些虚拟寄存器就会被映射到内存，这叫spilled
virtuals。

#### LLVM中寄存器如何表示

LLVM中物理寄存器用1\~1023范围的整数来表示，可以阅读GenRegisterNames.inc文件来查看具体某个架构是怎么编号的，比如查阅lib/Target/X86/X86GenRegisterInfo.inc就可以直到EAX寄存器编号为43，MMX寄存器MM0编号为65。

一些架构含有物理位置相同的寄存器，如X86架构的EAX, AX, AL共享了前8bit。这些物理寄存器在LLVM中标记为别名。可以查阅RegisterInfo.td文件来检查某个架构的哪些寄存器是别名。而且，MCRegAliasIterator类可以枚举出跟某一个寄存器别名的所有物理寄存器。

物理寄存器在LLVM中被分组为寄存器类。同一个寄存器类中的寄存器是功能等价的，可以互换使用。每个虚拟寄存器只能被映射到某个寄存器类中的物理寄存器。例如，X86上，一些虚拟寄存器只能分配到8bit寄存器中。寄存器类用TargetRegisterClass对象描述，要想检查一个虚拟寄存器跟物理寄存器是否兼容，可以使用以下代码：
```c++
bool RegMapping_Fer::compatible_class(MachineFunction &mf,
                                      unsigned v_reg,
                                      unsigned p_reg) {
  assert(TargetRegisterInfo::isPhysicalRegister(p_reg) &&
         "Target register must be physical");
  const TargetRegisterClass *trc = mf.getRegInfo().getRegClass(v_reg);
  return trc->contains(p_reg);
}
```

有时为了调试需求，需要改变目标机可用的物理寄存器数量。这项任务必须在TargetRegisterInfo.td中静态完成，RegisterClass的最后一个参数就是寄存器列表，如果需要避免使用某些寄存器，就从这里注释掉。

虚拟寄存器也是用整数表示的。与物理寄存器不同，不同的虚拟寄存器从来不会共享编号。物理寄存器是在TargetRegisterInfo.td中事先定义好的，不能由应用开发者创建，而虚拟寄存器不是这样。为了创建新的虚拟寄存器，可以使用MachineRegisterInfo::createVirtualRegister()方法，它会返回一个新的虚拟寄存器。用IndexedMap<Foo, VirtReg2IndexFunctor>来容纳每个虚拟寄存器的信息。如果需要枚举所有的虚拟寄存器，就使用TargetRegisterInfo::index2VirtReg()方法来找到虚拟寄存器编号：
```c++
for (unsigned i = 0, e = MRI->getNumVirtRegs(); i != e; ++i) {
  unsigned VirtReg = TargetRegisterInfo::index2VirtReg(i);
  stuff(VirtReg);
}
```

在寄存器分配之前，虽然物理寄存器也有使用，不过指令操作数大部分都是虚拟寄存器。为了检查给定的操作数是否是寄存器，可以用MachineOperand::isRegister()方法。为了获得寄存器的整数编号，使用MachineOperand::getReg()。指令可能定义或使用寄存器。比如ADD reg:1026 := reg:1025 reg:1024使用了寄存器1024和1025，而定义了1026。给定寄存器操作数，方法MachineOperand::isUse()返回该寄存器是否被使用，而MachineOperand::isDef()返回该寄存器是否被定义。

我们将寄存器分配之前存在于LLVM IR中的物理寄存器叫做pre-colored寄存器。这类寄存器用于很多场景，比如，函数调用的参数传递，存储特殊指令的结果。有两类pre-colored寄存器：隐式定义的，显式定义的。后者就是正常的操作数，可以通过MachineInstr::getOperand(int)::getReg()来访问。而要想访问前者，就用TargetInstrInfo::get(opcode)::ImplicitDefs，其中opcode就是该指令的操作码。显式和隐式物理寄存器的重要区别就是后者是每条指令静态定义的，而前者会依赖具体被编译的程序而变化。例如，函数调用总是隐式定义或使用相同的物理寄存器集合。Pre-colored寄存器会给寄存器分配算法施加约束，寄存器分配器必须确保它们在活跃期间都不能被虚拟寄存器中的值所覆盖。

#### 把虚拟寄存器映射为物理寄存器

映射有两种方式：一是直接映射，使用TargetRegisterInfo和MachineOperand类中的方法；二是间接影射，依赖VirtRegMap类来插入load/store指令。

直接映射给寄存器分配器的开发者提供了更多灵活性，但是，也更容易出错，并且需要更多的实现工作。程序员需要指明在哪里插入laod和store指令。为了将某个物理寄存器指派给某个虚拟寄存器，用MachineOperand::setReg(p_reg)，插入store指令用TargetInstrInfo::storeRegToStackSlot(...)，插入load指令用TargetInstrInfo::loadRegFromStackSlot。

间接映射使开发者免于插入load/store指令的复杂性。用VirtRegMap::assignVirt2Phys(vreg,
preg)将虚拟寄存器映射为物理寄存器，用VirtRegMap::assignVirt2StackSlot(vreg)将虚拟寄存器映射到内存，并返回映射的那个stack slot位置。如果需要将另一个虚拟寄存器映射到相同的stack
slot，要用VirtRegMap::assignVirt2StackSlot(vreg, stack_location)。需要注意的一点是，当使用间接映射时，即便虚拟寄存器映射到了内存，仍然需要将它映射到物理寄存器，在对应值store之前，或reload之后，这个物理寄存器就是保存该值的位置。

如果使用间接映射，在虚拟寄存器映射到物理寄存器或栈槽之后，还需要使用spiller对象防止load/store指令。每个被映射到栈槽的虚拟寄存器在被定义之后都需要store到栈槽中，并在使用之前load回来。spiller的实现会尝试回收利用load/store指令，从而避免冗余。

#### 处理二地址指令

LLVM机器码指令一般都是三地址指令，也就是说，最多定义一个寄存器，使用两个寄存器。但是有些架构使用二地址指令，被定义的寄存器同时也是被使用的寄存器中的一个，例如，X86中的ADD %EAX, %EBX表示%EAX = %EAX + %EBX.。

为了生成正确代码，LLVM必须将这类指令转化为二地址指令。LLVM提供了TwoAddressInstructionPass
来处理这种情况，且需要在寄存器分配前调用。调用之后生成的代码不再是SSA形式了，比如，%a = ADD %b %c会被转化为如下代码：
```
%a = MOVE %b
%a = ADD %a %c
```

注意，在LLVM内部，第二条指令被表示为 ADD %a[def/use] %c。

#### SSA销毁过程

在寄存器分配过程中发生的一个重要转换是SSA的销毁。SSA形式简化了CFG上执行的很多分析，但是传统指令集是不实现PHI指令的，因此，为了生成可执行代码，编译器必须将PHI指令替换为等价的其他指令。

消除PHI指令有多种方式，最传统的一种就是用copy指令替换PHI，这也是LLVM采取的策略。SSA销毁算法实现在lib/CodeGen/PHIElimination.cpp中。为了调用该Pass，需要打上PHIEliminationID标识符作为标记，因为寄存器分配器需要它。

#### 指令折叠

指令折叠是在寄存器分配期间删除冗余copy指令的优化。比如，以下指令：
```
%EBX = LOAD %mem_address
%EAX = COPY %EBX
```

可以被安全替换为如下单条指令：
```
%EAX = LOAD %mem_address
```

通过TargetRegisterInfo::foldMemoryOperand(\...)方法可以将指令折叠。不过指令折叠时需要小心，折叠后的指令可能跟原始指令有相当大的差别。

#### 内置寄存器分配器

LLVM基础设施提供了四种不同的寄存器分配器：

- Fast --- 是debug构建中默认使用的分配器。在基本块级别分配寄存器，尽量将值保存在寄存器中。

- Basic --- 一种增量的寄存器分配方法。通过启发式算法给寄存器指派活跃范围，每次一个。由于分配期间代码可能被改写，该框架允许分配器作为扩展实现。这并不是一个生产级别的分配器，但是在修复bug时很有用，而且可以作为性能基准。

- Greedy --- 默认的分配器。在Basic分配器基础上做了高度调优，引入了全局活跃范围分割。该分配器致力于使溢出代码的代价最小化。

- PBQP --- 基于PBQP的分配器。该分配器将寄存器分配问题建模为PBQP问题，然后使用PBQP求解器解决它，再把答案映射为寄存器的指派方案。

llc使用的寄存器分配器的类型可以在命令行用-regalloc=...选项指定：
```bash
$ llc -regalloc=linearscan file.bc -o ln.s
$ llc -regalloc=fast file.bc -o fa.s
$ llc -regalloc=pbqp file.bc -o pbqp.s
```

### Prolog/Epilog代码插入
TODO

### Compact Unwind

TODO:

### 晚期机器码优化
TODO

### 代码输出

代码输出负责将代码生成器中的抽象表示（MachineFunction, MachineInstr等）lower到MC层使用的抽象表示（MCInst, MCStreamer等）。几个不同类合作完成这项任务：机器无关的AsmPrinter类，机器相关的AsmPrinter的子类，以及TargetLoweringObjectFile类。

MC层属于obj文件的抽象，不再有函数、全局变量等概念。相反，它会考虑标号，directive，指令等概念。此时使用的关键类是MCStreamer，它是可以用多种方式实现（输出.s或.o）的抽象API。MCStreamer针对每个directive有一个方法，比如EmitLabel, EmitSymbolAttribute, switchSection等，这都跟汇编级的directive是对应的。

如果对为某个机器实现代码生成器感兴趣，那么徐璈实现三件事情：

1. 首先，需要继承AsmPrinter。该类将MachineFunction向下转换为MC标号。AsmPrinter基类提供了一系列有用的方法和例程，并允许覆盖其向下转换过程。如果想实现ELF，COFF，或者MachO格式的机器，可以复用很多已实现的代码，因为TargetLoweringObjectFile类已经实现了大部分公共逻辑。

2. 其次，需要为你的目标机实现指令打印器。指令打印器以MCInst为输入，将其渲染成文本输出到raw_ostream。这里大部分都可以从.td文件中自动生成，但是仍然需要实现打印操作数的例程。

3. 再次，需要实现从MachineInstr到MCInst的向下转换，常常实现在\<target\>MCInstLower.cpp文件中。这个向下转换过程经常是目标机相关的，负责将跳转表条目、常量池索引、全局变量地址转换为MCLabels。该转换层负责将代码生成器使用的微操作扩展为对应的实际机器指令，生成的MCInsts会交给指令打印器或者编码器。

最后，根据你的选择，你也可以为MCCodeEmitter实现一个子类，将MCInst向下转换为机器码字节和重定位。如果你想要支持.o文件输出，或者想实现一个汇编器，这一步就很重要。

#### 输出函数栈大小信息

当TargetLoweringObjectFile::StackSizesSection非空，且设置了TargetOptions::EmitStackSizeSection (-stack-size-section)时，包含函数栈大小元信息的section就会输出。该section包含一个数组，其元素是函数符号值和栈大小组成的有序对。当然栈大小仅包含在函数prologue中申请的栈空间，不包括动态栈申请。

### VLIW打包器

在超长指令字架构上，编译器负责将指令映射到硬件功能单元上。为此，编译器会创建叫做packets或bundles的指令组。LLVM中的VLIW打包器就是完成机器指令打包的机器无关机制。

#### 从指令映射到功能单元

典型的VLIW机器指令会映射到多个功能单元。在打包过程中，编译器必须能够推理出一条指令是否能添加到指令包中。由于编译器需要检查所有可能的映射，这个决策过程可能会很复杂。因此，为了缓解这种复杂性，VLIW打包器会解析目标机的指令类，并在编译器构建期间生成表。可以通过机器无关的API查询这些表，来决定一条指令是否要容纳进一个指令包中。

#### 打包表如何生成和使用

打包器从目标机的itinerary描述中读取指令类，构建一个DFA来表示指令包的状态。一个DFA包含三个主要元素：输入、状态、转换。DFA的输入集合表示正在被添加到指令包中的指令，状态表示指令所消耗的可能的功能单元。在DFA中，从一个状态到另一个状态的转换发生在给已存在的指令包添加指令时。如果从功能单元到指令的合法映射存在，那么DFA就包含相应的转换。如果转换不存在，就意味着合法的映射不存在，指令不能添加到指令包中。

为了给VLIW机器生成表，需要在Makefile中将*Target*GenDFAPacketizer.inc添加为目标。导出的API提供了三个函数：DFAPacketizer::clearResources(), DFAPacketizer::reserveResources(MachineInstr *MI), 和 DFAPacketizer::canReserveResources(MachineInstr *MI).。这些函数允许打包器将一条指令添加到已存在的指令包中，以及检查指令是否能被添加到指令包中。

## 实现独立汇编器

LLVM支持实现完整的独立汇编器。其大部分代码都是从td文件中生成的。

### 指令解析

### 指令别名处理

指令解析一旦完成，就会进入MatchInstructionImpl函数。该函数做别名处理然后做实际的匹配。

别名处理是将相同指令的不同文本形式处理为一种标准形式。可能实现的别名有多种，下面按照它们被处理的顺序列出来了。通常情况下，你会想用第一种别名机制来满足实际需要，因为它允许更简洁的描述。

#### 助记符别名

别名处理的第一步是简单的指令助记符重映射，它就是一种简单的从一种输入助记符到输出助记符的无条件重映射。这种别名不会去看操作数，所以这种重映射必须对给定的助记符的所有形式都适用。助记符别名的定义是很简单的，比如X86如下：
```
def : MnemonicAlias<"cbw",     "cbtw">;
def : MnemonicAlias<"smovq",   "movsq">;
def : MnemonicAlias<"fldcww",  "fldcw">;
def : MnemonicAlias<"fucompi", "fucomip">;
def : MnemonicAlias<"ud2a",    "ud2">;
```

有了这类定义，助记符就可以简单直接地重映射。虽然助记符别名不会看指令内部信息，但是它们会通过Requires子句依赖于全局模式：
```
def : MnemonicAlias<"pushf", "pushfq">, Requires<[In64BitMode]>;
def : MnemonicAlias<"pushf", "pushfl">, Requires<[In32BitMode]>;
```

该例子中，根据当前指令集，助记符被映射到不同的指令。

#### 指令别名

别名处理的最一般步骤发生在匹配时：它提供了匹配器的新形式来匹配特定的指令生成。指令别名有两部分：要匹配的字符串和要生成的指令。例如：
```
def : InstAlias<"movsx $src, $dst", (MOVSX16rr8W GR16:$dst, GR8  :$src)>;
def : InstAlias<"movsx $src, $dst", (MOVSX16rm8W GR16:$dst, i8mem:$src)>;
def : InstAlias<"movsx $src, $dst", (MOVSX32rr8  GR32:$dst, GR8  :$src)>;
def : InstAlias<"movsx $src, $dst", (MOVSX32rr16 GR32:$dst, GR16 :$src)>;
def : InstAlias<"movsx $src, $dst", (MOVSX64rr8  GR64:$dst, GR8  :$src)>;
def : InstAlias<"movsx $src, $dst", (MOVSX64rr16 GR64:$dst, GR16 :$src)>;
def : InstAlias<"movsx $src, $dst", (MOVSX64rr32 GR64:$dst, GR32 :$src)>;
```

这展示了一个强大的指令别名例子，根据汇编中存在哪些操作数，以多种不同的方式匹配相同的助记符。指令别名的结果可以包含以不同于目标指令的顺序排列的操作数,并且可以多次使用同一个输入，例如：
```
def : InstAlias<"clrb $reg", (XOR8rr  GR8 :$reg, GR8 :$reg)>;
def : InstAlias<"clrw $reg", (XOR16rr GR16:$reg, GR16:$reg)>;
def : InstAlias<"clrl $reg", (XOR32rr GR32:$reg, GR32:$reg)>;
def : InstAlias<"clrq $reg", (XOR64rr GR64:$reg, GR64:$reg)>;
```

这个例子也展示了绑定的操作数只列出一次。在X86后端,XOR8rr有两个输入GR8和一个输出GR8(其中一个输入与输出绑定)。InstAliases 获取一个简化的不重复的操作数列表，指令别名的结果也可以使用立即数和固定的物理寄存器,它们在结果中会被添加为简单的立即数操作数，例如：
```
// Fixed Immediate operand.
def : InstAlias<"aad", (AAD8i8 10)>;

// Fixed register operand.
def : InstAlias<"fcomi", (COM_FIr ST1)>;

// Simple alias.
def : InstAlias<"fcomi $reg", (COM_FIr RST:$reg)>;
```

指令别名也可以使用Requires子句来使它们特定于具体的子目标机器。

如果后端支持，指令打印器还可以自动输出别名，而不是被别名的指令。通常这会导致代码更好更易读。如果确实需要被别名的指令，就在InstAlias定义时将0传入作为第三个参数。

### 指令匹配
TODO


---

> 作者: [lazypanda](https://github.com/wanghuibin0)  
> URL: https://simplecoding.fun/posts/2024/02/llvm-code-generator-1/  

