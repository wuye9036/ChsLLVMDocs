#LLVM中目标无关(Target-Indepenent)的代码生成(Code Generator)

* [译序](#prolugue)
* [介绍](#introduction)
* [描述目标平台的类](#tardesc_classes)
* [描述机器码的类](#mcdesc_classes)
* [MC层](#mclayer)
* [目标无关的代码生成算法](#cgalgo)
* [Native Assembler的实现](#nativeassembler)
* [特定平台(Target-Specific)的一些注意事项](#targspec_notes)

----------------------------

<h2 id=“prologue”>译序</h2>

LLVM的名字其实弱爆了，Low Level Virtual Machine。从字面上看去它不过是一个类似于JVM的虚拟机。但是实际上这个项目自从被苹果包养之后，它已经远远超出了一个虚拟机所具有的能力。其实我觉得它压根儿就和虚拟机没什么关系，完全就是一个Compiler。

特别是2010年前后LLVM自举（就是自己编译自己，是编译器一个非常重要的里程碑）之后，很快就被集成进XCode中取代了原先的GCC，这也使得它的曝光度大增。

LLVM是一个大的项目，有一系列的子项目如：

* Core: LLVM IR的分析，转换，优化与编译（也是狭义上的LLVM）
* Clang: "LLVM Native"的C/C++/Objective-C编译器前端（输出就是LLVM IR）
* Dragonegg: GCC 4.5的编译器前端（输出也是LLVM IR）
* LLDB: 类似于GDB的Debugger

当然还有其他诸如Polly一类优化器或者libc++这样的基础库项目。

做一个编译器有很多路子。解释器也好，一遍编译也好，都算是编译器的Framework。但是随着语言机制的愈发复杂以及后端技术上的日益完善，编译器中用于语法分析和语义处理的前端和用于生成目标代码的后端的界限也越来越清晰。某种意义上，借助于中间语言（Intermediate Language，IL）或者称中间表达（Intermediate Representation, IR）的方法已经成为产品级编译器开发中绝对主流的技术。

中间语言/中间表达是个很宽泛的概念。它可能是抽象语法树，也可能是SSA Form，或者是MSIL这样的伪汇编。当然汇编从学理上也可以作为IR，只不过不管是哪个平台的汇编，都和硬件关系紧密（恐怕就MMIX会好一些，所以也只有Knuth这个全球唯一的CPU），它们作为IR会带来很多的复杂度。这也正是为什么要有一个高层抽象的IR在汇编和编程语言之间，编译器把众多的前端语言（C，C++，Objective C，C#）编译成IR，再在IR上进行优化，最后再编译到多个平台（x86，x86-64，PowerPC，etc）上。

LLVM的IR类似于汇编，如你看到的这样。

	define i32 @add(i32 %a, i32 %b) {
	entry:
	  %tmp = add i32 %a, %b 
	  ret i32 %tmp
	}

但同时，LLVM的IR是一种SSA Form的IR。所谓SSA，即Static Single Assignment，称为 *[静态单赋值](http://zh.wikipedia.org/zh-cn/%E9%9D%99%E6%80%81%E5%8D%95%E8%B5%8B%E5%80%BC%E5%BD%A2%E5%BC%8F)* 。好处什么的这里不扯远，它使得LLVM IR有个重要的限制，每个值只能在定义(defined)时被赋值，之后再也不能被改变。比方说

	%3 = fadd %1, %2 
	%5 = fadd %3, %4 

就是一个可以被接受的结果，而

	%3 = fadd %1, %2
	%3 = fadd %3, %4 

就是错的，因为`%3`一旦被定义就不能被再次赋值了。这也是 *静态单赋值* 这个名字的由来。

当然，单有这个限制是没法写程序的。之所以这样做，是因为静态单赋值极大地解除了数据流分析和控制流分析的复杂度。这两类分析对于编译器优化和静态检查是关键性的技术。但是这样连输入输出都没法处理。这也是函数式编程会有Monad这样的方案的原因之一。[这里](http://www.lingcc.com/2011/08/13/11685/)是一篇单静态赋值的简介，可以解决一部分上面所提到的问题。实践中的IR是如何解决这些问题的，可以参考[LLVM Tutorial](http://llvm.org/docs/tutorial/)。

从静态单赋值形式的IR到最终的汇编，有很长的路要走。你需要选择合适的机器指令；将IR中支持的相对复杂的数据类型运用各种神通转换成机器可以直接执行的数据类型；物理寄存器的分配；以及最重要、也是很多人最常挂在嘴边但是又不太了解的：编译器优化。

而此篇文档，正是描述了这些技术在LLVM中的实现。

废话结束，正篇开始。

-----------------------------------------------------

> 译注：本文在说 _目标平台_ 时对应的原文中的单词是 _Target_ 。为了简单起见，在不产生歧义的情况下，我会将 _Target_ 翻译成 _平台_ 。

<h2 id = "introduction">介绍</h2>
LLVM所提供的平台无关的代码生成器，实际上是一个Framework。它给提供了一些可以复用的组件，帮助用户将LLVM IR编译到特定的平台上。当然，编译目标是很灵活的，它既可以是文本形态的汇编也可以是二进制的机器码。前者一般用于静态编译器，后者则可以用于JIT。整个代码生成由以下六个部分构成：

1. **描述平台特性的抽象接口（Abstract Target Description Interfaces）** 定义了一组用于描述平台特性的接口。这组接口仅用来说明平台的特性，至于它怎么被使用，接口本身并不关心。所有的接口都可在`include/llvm/Target/`中见到。   
2. 一组表达 **生成后代码（Code being generated）** 的类。这些类保存的并不是平台相关的指令和数据，而是一些在任意平台上都有效的概念`（译注：也包括在所有平台上都能直接或间接支持的指令，比方说ADD, SUB。虽然它们在不同平台上实现完全不同。）`。例如 _常量池项（Constant Pool Entries）_ 和 _跳转表（Jump Tables）_ 这些都是在这一层体现的。代码见`include/llvm/CodeGen/`。   
3. **MC Layer中所需要的类和算法** 。这些类和算法用来生成object file中的代码。它们能表达汇编中的内容，比方说 _标识（Label）_ ， _节（Sections）_ 和 _指令（Instructions）_ 。 _跳转表_ 这些东西在这一层就已经没有了。  
4. 生成Native code时使用的 **平台无关的算法（Target-independent Algorithm）** 。例如 _寄存器分配（Register Allocation）_ ， _指令调度（Scheduling）_， _栈帧的表达（Stack Frame Representation）_ 这些，都是平台无关的。代码见`lib/CodeGen`。  
5. 第五个部分是 **针对特定平台实现的平台描述接口（Implementation of the abstract target description interfaces）** 。LLVM的代码会用到这些描述，用户也能通过它专为某个平台提供特定的Passes。这些Passes和LLVM的代码一起，构成一个完整的针对特定平台的代码生成器。具体代码参见`lib/Target/`。
6. 最后一个部分是 **平台无关的JIT组件** 。LLVM JIT本身是完全和平台没有关系的。不过有一个接口`TargetJITInfo`，可以帮助JIT处理一些平台相关的问题。JIT的平台无关部分的代码，在`lib/ExecutionEngine/JIT`。 

介绍完组件后，你可以根据自己的兴趣和需要，阅读不同的章节。一般来讲我们建议你最好熟悉一下 _平台描述（Target Descrption）_ 以及 _机器码表达(Machine Code Representation)_ 相关的内容。如果想给新平台写一个后端，那么一方面你得知道怎么去 _实现自己的平台描述_，另一方面你也需要知道 _LLVM IR是怎么写的_ 。如果你只是单纯的想实现一个新的 _代码生成算法_ ，那么你只要在 _平台描述_ 和 _机器码表达_ 相关的类里面折腾就行了。当然你得保证你的算法是可移植的。

<h3>代码生成的必要组件			 </h3>
整体上LLVM的代码生成器可以看做是两个部分，一个用于代码生成的高层接口，一个是用来为开发后端的一组可复用组件。对于用户自定义的后端，最少情况下，只需要实现 _TargetMachine_ 和 _DataLayout_ 两个接口，就可以LLVM协同工作了。当然如果后端还需要使用到代码生成器中的一些组件，那还需要实现其他的接口。

这个设计有两个目的。一来这样做LLVM就可以支持非常特殊的目标平台。比如若编译目标是C语言，那诸如寄存器分配，指令选择这些传统的代码生成组件统统都不需要了。你只要实现了两个必须实现的接口，那它就能和LLVM一起工作。再比如，用户需要开发一个后端，将LLVM IR编译成GCC RTL，那也可以只实现这两个必须的接口。

二来，它允许用户开发全新的、不依赖任何现有组件的代码生成器，并集成到LLVM中。当然大部分情况下，我们还是建议能复用就复用。但是目标平台如果太奇葩，没办法用LLVM的模型来描述（do not fit into the LLVM machine description model），例如给FPGA开发后端，这个设计就起作用了。

<h3>代码生成器的高层设计			 </h3>
LLVM中使用的平台无关的代码生成器，是针对标准的寄存器机设计的。同时它对效率和生成代码的质量也有一定的要求。整个代码生成可以分为以下阶段：

1. **指令选择（Instruction Selection）** — 这个步骤的作用是将LLVM IR转化成目标平台的指令集。不过这个阶段产生的指令还是很原始的。它一方面借助 _虚拟寄存器（Virtual Register）_ 的概念，让目标代码仍然是SSA Form的；另一方面，为了满足平台的限制和调用协议的需要，它也使用了部分的 _物理寄存器（Physical Register）_ 。最终，LLVM IR被转化成一个由目标平台的指令所组成的 _DAG_。
2. **调度与整理（Scheduling and Formation）** — 耳熟能详的指令重排就是这个部分的主要工作。这个阶段将会读取DAG，并根据需要进行重新排列，最后将指令以`MachineInstr`s输出。不过虽然我们这里是单独列为一个阶段，但是实际上它和 _指令选择_ 都操作的是 _SelectionDAG_，并且两者的实现也极为接近，所以在讲算法的时候，将它和 _指令选择_ 放在一起讨论。
3. **基于SSA的优化** — 在经历了指令选择和调度后，接下来就需要优化上一阶段输出的DAG。整个优化过程实际上是有很多个基于SSA的优化步骤所组成`（译注：这也是为什么要在指令选择以后仍然保持SSA Form的原因之一。）`。比方说 _模调度（modulo-scheduling）_ 和 大名鼎鼎的 _窥孔优化（peephole optimization）_ 都是可以在这个阶段执行的。
4. **寄存器分配** — 到目前为止，我们指令的操作对象，都是在虚拟寄存器和部分的物理寄存器上的。LLVM的虚拟寄存器机制，可以看作是一个无穷大的 _虚拟寄存器文件（Virtual Register File）_。但在硬件上，指令只能去操作有限的物理寄存器，或者是内存地址。这个时候我们需要借助 _寄存器溢出（Spilling）_ 的办法，将原本是虚拟寄存器的指令参数，都落实到物理寄存器或内存地址上。
5. **Prolog/Epilog的生成** — 当函数体的指令都生成后，就能够确定需要的堆栈大小了。这个时候我们就需要在函数前后安插一些Prolog和Epilog的代码用于分配堆栈，同时原先没有确定的堆栈位置在此时也可以算出准确的偏移。知道了这些信息，我们还可以完成_栈帧指针消除（Frame Pointer Elimination）_ 或 _堆栈打包（Stack Packing）_ 这一类的优化。
6. **机器码的晚期优化** — 这个阶段可是最后一次给低效的代码卖后悔药了。基本上到了这里的代码已经和“最终”代码相差无几。不过你仍然有机会进行一些 _Spill code scheduling_ 或者是 _窥孔_ 方面的优化。
7. **代码发射（Code Emission）** — 在这一步你就需要输出调整完毕的代码了。一般输出代码都以函数为单位。当然目标格式没有什么限制，你可以选择是汇编或者是二进制的机器码。

之所以LLVM的代码生成采用这样的设计，是基于一个重要假设：我们只需要用一个_最优模式匹配选择子（Optimal Pattern Marching Selector）_ 来匹配生成目标代码，就能够获得较高质量的本地指令。当然你说有更牛逼的设计，比方说 _模式展开（Pattern Expansion）_ 或者 _激进迭代窥孔优化（Aggressive Iterative Peephole Optimization）_ ，但是他们实在是太慢了，特别是对于JIT而言。而且这个设计也可以通过多趟优化，来满足普通编译器激进优化的需求。

上面这些步骤大部分都是平台无关的，但是后端完全可以加入任何平台相关的处理过程。举个栗子！x86体系中，x87浮点运算绝对是个傲娇。它非得要 _浮点堆栈（Floating Point Stack）_ 来搞浮点运算。所以LLVM在x86的实现中就专门有一个Pass来处理这个问题。所以如果你在其它平台遇到了类似的需求，也可以如法炮制。

<h3>使用TableGen生成目标平台描述	 </h3>
用来描述平台特性的类（Target Description Classes）需要存储并提供大量关于目标架构的细节信息。仔细观察会发现，这当中很多信息是相同的。例如`add`所需要的信息几乎和`sub`是完全一致。为尽可能复用这些共性，LLVM使用了一个DSL工具`TableGen`来描述目标平台的信息。这个DSL工具可以更好的抽象出平台上的特性，并尽可能的减少重复的代码。

未来LLVM的开发计划中，我们希望能够把所有的平台描述都移到`.td`文件中。这么做最重要的原因是我们希望LLVM移植起来更方便，因为C++的代码少了，而且移植它的程序员要理解的内容也会少很多。另外，这也会让LLVM更容易修改。如果所有的工作都是`tblgen`完成的话，一旦接口有调整那也只需要修改`tblgen`的实现就可以了。

<h2 id = "tardesc_classes">用于描述平台的类</h2>
在LLVM中，与平台描述相关的一组类为不同的平台提供了一个相同的抽象接口。这些类在设计上仅仅用来表达目标平台的属性，例如平台所支持的指令和寄存器，但是不会保存任何和具体算法相关联的描述。

所有的平台描述类，除了`DataLayout`外都是可以被继承的。用户可以根据不同的平台提供具体的子类，这些子类通过重载接口的虚函数来提供平台信息。`class TargetMachine`提供了一组接口，可以访问这些描述。

<h3>class <code>TargetMachine		</code></h3>
`class TargetMachine` 有很多的虚方法可以获得具体的平台描述。这组函数一般都命名成`get*Info`，（如 `getInstrInfo`, `getRegisterInfo`, `getFrameInfo`）。`TargetMachine`也是通过继承为接口提供平台相关的实现（如 `X86TargetMachine`）。如果你只是想实现一个能被LLVM支持的最简单的`TargetMachine`，那只要能返回`DataLayout`就可以了。如果你使用了LLVM其他的代码生成组件，那就需要实现诸如`getInstrInfo`等其他的接口函数。

>	译注：这里可以看做是一个抽象工厂模式。
>	`TargetMachine`和`Target Description Classes` 的关系，实际上是就抽象工厂和抽象工厂的抽象产品的关系。
>	X86TargetMachine就是具体工厂，而例如X86RegisterInfo就是具体工厂的具体产品。

<h3>class <code>DataLayout			</code></h3>
所有平台描述类中，只有`DataLayout`是必须要支持的，同时它也是唯一一个不能继承的类。结构体中成员的内存布局、不同类型的数据对内存对齐的要求、还有平台上指针的大小、平台是Little Endian还是Big Endian，这些信息都保存在DataLayout中。

<h3>class <code>TargetLowering		</code></h3>
（译注：Lowering是指高层指令向底层指令转化的过程）`TargetLowering`用于指导LLVM的指令选择模块如何将LLVM IR指令转化成SelectionDAG上的节点和操作。该类型保存了以下的信息：

* 对寄存器进行分类，以匹配不同的`ValueType`s（an initial register class to use for various `ValueType`s），
* 哪些操作（指令）是平台原生支持的，
* `setcc`的返回类型，
* _偏移量(shift amounts)_ 的类型，
* 其他的高层特性，例如是否要把除以常量转换成一个乘法。

<h3>class <code>TargetRegisterInfo	</code></h3>
`TargetRegisterInfo`描述了目标平台的寄存器文件以及寄存器间的交互。

在代码生成中，每个寄存器都会给一个无符号整数的编号。物理寄存器（也就是平台实际支持的寄存器）的标号通常都较小，而虚拟寄存器的序号一般都往大了标。不过`#0`寄存器被LLVM保留下来当标志寄存器用了。

每个寄存器都对应一个`TargetRegisterDesc`项，这个项保存了寄存器的文本名称（比方说`EAX`，`ECX`之类，输出汇编和dump要用到），以及一坨 _寄存器别名（register alias）_ （如果有的话，寄存别名是说两个寄存器占用了同样的空间，比方说x86中的`EAX`和`AX`）。

`TargetRegisterInfo`还提供了处理器支持的寄存器类别（保存在`TargetRegisterClass`的实例中）。每一类都包含了若干相同属性的寄存器（比方说`EAX`、`EBX`、`ECX`、`EDX`都是32位的整数寄存）。虚拟寄存在创建的时候，也会根据要保存的数据类型，关联到对应的寄存器类别上。所以虚拟寄存器在最终被替换成物理寄存器的时候，也是使用相同类别的寄存器进行替换。

基本上本节涉及类型的具体类一般都是由`.td`直接生成的。

<h3>class <code>TargetInstrInfo		</code></h3>
`TargetInstrInfo`是用来描述平台指令。基本上它就是一个`TargetInstrDescriptor`的数组。后者望文生义，就是保存了单条指令的信息，常见的有指令的助记符， _参数（Operands）_ 的数量，一些隐含的寄存器使用与定义。指令的信息里既有平台无关的部分，例如是否操作了内存(accesses memory)，或者能交换（is commutable），也会有平台相关的标记。

<h3>class <code>TargetFrameInfo		</code></h3>
`TargetFrameInfo`提供了关于堆栈布局的信息，包括堆栈的增长方向，堆栈的对齐要求以及 _本地偏移（offset to the local area）_。这里的 _本地偏移_ 是指函数内的数据（局部变量或者Spilling Location）的起始地址到函数入口处的偏移量。

<h3>class <code>TargetSubtarget		</code></h3>
`TargetSubtarget`保存了一些关于和平台具体实现（例如芯片组）有关的信息。一般来说，`Subtarget`会支持或禁止一些指令，指令有具体的延迟时间或者是有特别的指令执行顺序，`TargetSubtarget`需要反映这些信息。

<h3>class <code>TargetJITInfo		</code></h3>
如果`TargetMachine`需要支持JIT的话，那么这个`TargetMachine`就需要实现方法`getJITInfo`，它将返回一个由`TargetJITInfo`派生出的具体类型的实例。JIT过程中需要的一些行为，都需要由`TargetJITInfo`的派生类提供支持，例如怎么去生成一个 _桩函数（Stub）_。(译注：Stub是JIT中一个很重要的概念。在JIT程序初始化的时候，会将每个函数都初始化成Stub。这个Stub很短很简单，他的逻辑是，当自己被调用的时候，就去找JIT Engine，为自己生成一个真正的函数体，然后把自己替换掉，以执行真正的逻辑。)

<h2 id = "mcdesc_classes">描述机器码的类</h2>
简单来说，从LLVM code生成的机器码最后会由以下三个类的实例来表达， `MachineFunction`, `MachineBasicBlock`和`MachineInstr`。相关的代码可以在`include/llvm/CodeGen/`中找到。每条指令由一个 _操作码（opcode）_ 和 若干个 _参数（Operands）_ 构成。很显然这个表示方法在任何平台上都是适用的，它既能将机器代码表现成SSA Form的形式，也能在寄存器分配完成后表现非SSA形式的汇编。

<h3>class <code>MachineInstr</code></h3>
`MachineInstr`是表现目标指令的基本单元。在设计上，它是机器指令的一个非常好的抽象，它仅仅保存了 _opcode_ 和 _operands_。 _操作码_ 是一个无符号整数，具体它代表什么的指令，是由相应平台的后端来解释的。所有的 _opcode_ 都在 _*InstrInfo.td_ 中定义， _opcode_ 的枚举值是由 _td_ 文件生成的。`MachineInstr`只存储 _opcode_，但是它不关心这个 _opcode_ 的含义是什么。指令代表的意义需要到`TargetInstrInfo`中去查询。

指令的 _operand_ 有多种类型，他们可能是一个寄存器（寄存器编号），一个常量（立即数）或者是一个基本块（如跳转指令的目标）。另外， _operand_ 需要标记为 _def_ 或 _use_ （译注：简单理解，def就是指令的输出，use就是指令的输入）。不过在LLVM中，只能将寄存器作为 _def_（译注：可能是出于SSA的考虑）。

在指令的参数里同时有 _def_ 和 _use_ 的时候，LLVM规定要将 _def_ 放在所有 _use_ 的前面。这点和平台是无关的。即便在SPARC这样的体系上，一条指令的得写成`add %src0, %src1, %dest`的形式，在LLVM中，三个参数仍然表示为`%dest, %src0, %src1`这样的顺序。

让 _目标操作数（destination or definition operands）_ 在最前面是有很多好处的，起码在调试时打印代码的时候你就能轻松打出

	%dest = add %src0, %src1

这般赏心悦目的代码（译注：真TM狡辩 ~）。当然，还有一个好处是在 _创建指令_ 的时候也会更加方便。（译注：我也觉得也是狡辩~）

<h4>使用<code>MachineInstrBuilder.h</code>中的函数</h4>
`include/llvm/CodeGen/MachineInstrBuilder.h`中有个函数`BuildMI`是专门用来帮助用户更加方便的生成指令。下面的代码是一个使用示例：

```C++ 
// Create a 'DestReg = mov 42' (rendered in X86 assembly as 'mov DestReg, 42')
// instruction.  The '1' specifies how many operands will be added.
MachineInstr *MI = BuildMI(X86::MOV32ri, 1, DestReg).addImm(42);

// Create the same instr, but insert it at the end of a basic block.
MachineBasicBlock &MBB = ...
BuildMI(MBB, X86::MOV32ri, 1, DestReg).addImm(42);

// Create the same instr, but insert it before a specified iterator point.
MachineBasicBlock::iterator MBBI = ...
BuildMI(MBB, MBBI, X86::MOV32ri, 1, DestReg).addImm(42);

// Create a 'cmp Reg, 0' instruction, no destination reg.
MI = BuildMI(X86::CMP32ri, 2).addReg(Reg).addImm(0);

// Create an 'sahf' instruction which takes no operands and stores nothing.
MI = BuildMI(X86::SAHF, 0);

// Create a self looping branch instruction.
BuildMI(MBB, X86::JNE, 1).addMBB(&MBB);
```

代码本身很容易懂。不过要注意的是，所有重载形式的`BuildMI`总有一个必要参数指定了 _operands_ 的数量。这是为了在给指令的分配内存的时候可以更加有效率。（译注：和vector.reserve的作用相同）。

另外，示例中的添加的 _operands_ 都是以 _use_ 的形式添加的。如果想要为指令指定一个 _def_ 语义的寄存器（也就是为指令指定一个输出），可以使用以下的形式：

	MI.addReg(Reg, RegState::Define);

<h4>固定（预分配）寄存器</h4>
在做代码生成的时候，都会遇到一个关键的需求，那就是在一些特定的情况下，需要给指令分配特定的寄存器。这个要求往往是特定的指令集的带来的。比如在x86下你想要执行整数除法，就必须要在`EAX`和`EDX`中加载相应的操作数。还有就是调用协议的限制`（译注：比如fastcall和x64下的calling convention）`。在这些场合中，必须要为指令生成正确的物理寄存器，并将数据拷贝到物理寄存器上（或者从物理寄存器拷贝到虚拟寄存器）。

再举个栗子！下面是一段简单的LLVM的代码：

	define i32 @test(i32 %X, i32 %Y) {
	  %Z = udiv i32 %X, %Y
	  ret i32 %Z
	}

把这段代码编译到x86上后，`div`的除数需要放到`EAX/EDX`中，而`ret`返回值则要放在`EAX`内。我们执行下`llc X.bc -march=x86 -print-machineinstrs`来看看LLVM是怎么处理的：

	;; Start of div
	%EAX = mov %reg1024           ;; Copy X (in reg1024) into EAX
	%reg1027 = sar %reg1024, 31
	%EDX = mov %reg1027           ;; Sign extend X into EDX
	idiv %reg1025                 ;; Divide by Y (in reg1025)
	%reg1026 = mov %EAX           ;; Read the result (Z) out of EAX

	;; Start of ret
	%EAX = mov %reg1026           ;; 32-bit return value goes in EAX
	ret

代码生成以后，寄存器分配阶段会合并寄存器（coalesce the registers）使用，也就是删掉一些等价的寄存器，生成以下代码：

	;; X is in EAX, Y is in ECX
	mov %EAX, %EDX
	sar %EDX, 31
	idiv %ECX
	ret

拷贝-合并这个方法非常通用（x86都搞的定了还能有啥搞不定的！），同时它还能让 _指令选择子（instruction selector）_ 在不知道平台细节的情况下顺利工作。不过有一点要注意，就是物理寄存器的 _生命期（Lifetime）_ 越短越好。`（译注：关于这个问题的简单理解，可以认为物理寄存器是稀少而高效的资源。每次使用它的时间越短，它就能在更多需要它的场合发挥作用。）`所以LLVM中假设物理寄存器的生命周期一定控制在 _Basic Block_ 内。所以如果要将值在 _Basic Block_ 之间传递， **必须** 使用虚拟寄存器。

<h4>call-clobbered寄存器</h4>
部分指令例如`call`，会在调用时 _篡改（clobber）_ 很多的寄存器`（译注：clobber的含义是在指令调用后，会使得很多寄存器失效。比方说call后的那些易失(volatile)寄存器）`。我们可以通过增加`<def, dead>`参数，记录哪些寄存器可能被篡改了。但是LLVM中的方法则是通过`MO_RegisterMask`参数，给所有 _被保留的寄存器（reserved register）_ 提供一个掩码，其它的寄存器则被认为是在调用时被修改了。

<h4>SSA form的机器码</h4>
从生成`MachineInstr`s开始，到 _寄存器分配_ 之前，`MachineInstr`一直都是以SSA-form存储的。我们将LLVM IR中的 _PHI_ 节点直接转换成机器码的 _PHI_ 节点， 所有的 _虚拟寄存器_ 也都是 _静态单赋值_ 的。

在 _寄存器分配_ 之后，机器码便不再以SSA-form存储。此时所有的虚拟寄存器都被移除或被物理寄存器替代，机器码的形式更加接近本地指令的形式。`（译注：当然PHI这么高级的指令通常也要被替换掉。）`

<h3>class <code>MachineBasicBlock</code></h3>
`MachineBasicBlock`可以看成是`MachineInstr`的列表。`MachineBasicBlock`基本上与LLVM IR是对应的。每一个IR中的`BasicBlock`都对应一个或多个`MachineBasicBlock`。每个`MachineBasicBlock`都可以通过`getBasicBlock`函数获得它所属的IR`BasicBlock`。

<h3>class <code>MachineFunction</code></h3>
`MachineFunction`与LLVM IR中的`Function`一一对应。每个IR`Function`中产生出的所有`MachineBasicBlock`s都以列表的形式存储在它对应的`MachineFunction`中。除了`MachineBasicBlock`s之外，每个`MachineFunction`还保存了`MachineConstantPool`，`MachineFrameInfo`和`MachineRegisterInfo`等信息。更多的细节可以参见`include/llvm/CodeGen/MachineFunction.h`。

<h3><code>MachineInstr Bundles</code></h3>
LLVM的代码生成还可以将一串指令搞成一个`MachineInstr` _Bundles_。MI _Bundle_ 主要是给VLIW一类的架构提供一个打包的指令组，一个包中的指令数量不限，指令之间是可以并行的。它也能用来表达一些必须整包执行的指令组`（译注：这个时候Bundle内的指令不一定能并行）`，比方说 _ARM Thumb2 IT Blocks_。

从概念上来讲MI _Bundle_ 应该是内嵌了一组MIs，比方说下图这个样子：

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

不过在LLVM中，我们为了不改变`MachineBasicBlock`和`MachineInstr`的设计，并没有采用这种内嵌分组的方案。即便MIs逻辑上是在一个 _bundle_ 中，我们也仍然把它和其他的普通指令存放在同一个列表中，只不过 _bundled_ 的指令会被标记成`InsideBundle`， _bundle_前也会有一个特殊的`BUNDLE`指令告诉你一个 _bundle_ 开始了。

> 译注：我对VLIW的结构不熟悉，也没有深究过Bundle的实现，所以这里在阅读的时候微有歧义。

一般在处理`MachineInstr`的时候，整个MI _bundle_ 是做为一条 _指令(unit)_ 来处理的。`MachineBasicBlock`中的`iterator`在遇到 _bundle_ 的时候，也是 _作为一个完整的单元(as-a-single-unit)_ 直接跳过。如果你想完全的按指令遍历，可以使用`instr_iterator`，此时它会历所有的指令，包括内嵌在 _bundle_ 中的。另外，作为 _bundle_ 起始的`BUNDLE`指令，需要标记出 _bundle_ 内的指令所有使用的输入和输出寄存器。

`MachineInstr`打包成 _bundle_ 的工作是在寄存器分配阶段完成的。具体来说，只有当所有SSA相关的工作，包括 _双址指令转换（Two-address pass）_、 _PHI节点清除（Phi Elimination）_ 以及 _合并拷贝操作（Copy coalescing）_ 完成后，才能确定哪些指令可以被打包到一起。在这之后，需要等虚拟寄存器到物理寄存器的转换完成之后，才能进行`BUNDLE`指令收尾工作，例如将指令添加到`BUNDLE`中，并设定`BUNDLE`指令的参数（寄存器）。之所以不放在虚拟寄存器的阶段是因为彼时`BUNDLE`指令也需要占用虚拟寄存器，它会导致虚拟寄存器的使用量加倍。

> 译注：这里举个例子，比方说有一组指令共占用了四个虚拟寄存器作为输入或者输出，那么bundle之后，BUNDLE指令也引用到这四个虚拟寄存器。虚拟寄存器中保存了引用或定义它的指令的列表，这样它不仅需要保存使用它的子指令，还兼带保存了BUNDLE指令。就使得这个列表变得更长了。特别是如果寄存器所牵涉到的指令都是在 _bundle_ 中的，那这样做就几乎使得这个列表翻倍了。

<h2 id="mclayer">The 'MC' Layer</h2>
_MC Layer_的功能基本上和汇编器差不多，像什么 _常量池_， _跳转表_， _全局变量_ 这些概念它统统都没有。它能处理的，就是类似于 _标号（Label）_， _机器指令_， _Object file的节（section）_ 这些更加底层的概念。它的设计目的很明确，就是把处理好的指令输出成`.s`或者`.o`文件，以及作为`llvm-mc`这个工具的后端来处理代码的汇编和反汇编。

<h3><code>MCStreamer</code> API</h3>
`MCStreamer`是一个非常棒的汇编器接口。针对不同的输出，例如`.s`或者输出`.o`，它都需要不同的实现。它的API与汇编文件的内容是对应的，它为每个汇编文件中的 _指示字（directive）_ 都提供了一个独立的函数，例如`EmitLabel`，`EmitSymbolAttribute`，`SwitchSelection`，`EmitValue`(`.byte`, `.word`)。还有`EmitInstruction`用于将`MCInst`添加到流中。

LLVM中有两个地方用到了这个接口，一个是汇编器`llvm-mc`，一个是 _代码发射（Code Emission）_ 阶段，`MC Layer`中会调用`MCStreamer`的接口把IR或机器码对象转换成`.s`或者别的什么。

实现方面LLVM提供了两个基本实现，`MCAsmStreamer`输出`.s`文件，`MCObjectStreamer`输出`.o`文件。相比之下前者的实现比较简单，只要把 _指示字_ 按照文本输出就行了，而后者则有完整的汇编器功能。

<h3>class <code>MCContext</code></h3>
所有在MC阶段需要使用的数据如 _符号_ 和 _节_ 什么的都放在`MCContext`中。所以你要创建符号或者节就得和它打交道。另外这个类型不能被继承。

> 译注：快看！果然Context才是王道！

<h3>class <code>MCSymbol</code></h3>
`MCSymbol`直接对应了汇编文件中的 _symbol_（或者叫 _label_）。我们将所有的符号分成两类，一类是临时符号，一类是常规符号。临时符号只是在汇编的时候会使用，等object file生成之后，这些符号就会被丢弃掉。临时符号会有特定的前缀进行区分，例如在 _MachO_ 上的前缀就是"L"。

`MCSymbol`有`MCContext`产生和管理，同时在`MCContext`中，每个Symbol只有一个实例。这样`MCSymbol`之间的比较用指针就可以了。不过不同的Symbol可能会指向相同的地址，比方说看下面这个例子：

	foo:
	bar:
	  .byte 4

在这种情况下`foo`和`bar`实际上都是指向`.byte 4`同一个位置的。

<h3>class <code>MCSection</code></h3>
`MCSection`是对object file中 _section_ 的描述。因为不同对象文件的实现中节的表示方法都不一样，因此对于不同的对象它都有不同的实现，如`MCSectionMachO`，`MCSectionCOFF`，`MCSectionELF`。和MCSymbol相同， _section_ 也是由产生并且在`MCContext`内是唯一的。

<h3>class <code>MCInst</code></h3>
`MCInst`是MC阶段的机器指令。它比`MachineInstr`简单多了。虽然它的结构也是平台无关的，但是它保存的 _指令码（opcode）_ 以及一组 `MCOperand` 却都是平台相关的数据。MCOperand是一个union，它包含以下三种数据之一：立即数、目标寄存器ID或者符号表达式（`Lfoo-Lbar+42`）。符号表达式在MC阶段是由一个叫`MCExpr`的类表示。

在整个MC Layer中，`MCInst`的地位很重要。无论是编码，或者打印输出，或者是汇编器和反汇编器的输入输出，它都是存储指令的唯一手段。

> 译注：写到这里废话两句。这个文档中很多地方都在强调平台无关，好像一旦相关了它就是标题党一样。总结一下，LLVM的代码在以下方面都是平台无关的：

> 1. 结构上大多是平台无关的，比方说不管什么什么平台，它的指令都可以描述成Opcode + Operands的结构。
> 2. 大多数优化是平台无关的。
> 3. 很多处理过程，如指令的生成（Instruction Selection）当中有很多步骤是平台无关的（这也是LLVM为什么信心满满的说未来要让所有的平台相关信息都可以写到`td`文件中）。但是这里的平台无关是有一些代价的，例如你需要给寄存器和指令提供通用而详尽的特性（用于模式匹配），尽管其中一部分描述在某些平台上是没有什么用。
> 4. 最后就是，LLVM将很多实现与接口剥离开，比如MCStreamer。这样几乎可以让所有的逻辑都是平台无关的。
> 5. 当然也可以雄辩说，这些实现都是针对寄存器机的共性，所以“平台无关”的说法都是扯淡。

<h2 id="gcalgo">目标无关的代码生成算法</h2>

> 译注：终于到翻译到算法了。

在 _代码生成器高层设计_ 一节我们已经展示了LLVM的工作流程。这一节主要是解释原理和代码生成器设计时的考量因素。

<h3>指令选择(Instruction Selection)</h3>
整个 _指令选择_ 阶段，就是要把LLVM的代码转化成平台相关的机器指令。关于这个问题有很多的现有方案，LLVM使用的是 _基于SelectionDAG的指令选择子（SelectionDAG based instruction selector）_。

大部分 _指令选择子_ 的代码都是由目标描述（`*.td`）文件直接产生的。我们当然希望以后`td`文件能包办一切，但是现阶段我们还需要为指令选择子提供一些自定义的C++实现。

<h4>SelectionDAGs简介</h4>
_SelectionDAG_ 是代码的一种抽象表达。这个结构有很很多优点。它适合与一些自动化指令选择算法协同工作，比方说基于动态规划的最优模式匹配（dynamic-programming based optimal pattern matching selectors）。对于代码生成，特别是指令的调度（instruction scheduling）它也是个非常易用的结构。另外，有很多底层但是平台无关的优化，可以很方便的应用到SelectionDAG上，当然这些优化需要平台提供相应的信息。

LLVM中的 _SelectionDAG_ 是一个以`SDNode`对象作为节点的 _有向无环图（DAG，Directed-Acylic-Graph）_。`SDNode`最重要的属性 _操作码（Operation Code，Opcode）_ 和 _操作数（Operands）_。前者用来表示Node是做什么的，后者是操作的参数。`include/llvm/CodeGen/SelectionDAGNodes.h`中提供了一些预定义的节点类型（Operation Node Types）。

每个Node都可能使用其它节点作为输入，`SDNode`的 _operands_ 就用来保存它们所引用到的`SDNode`s，这也称作是 _边（Edges）_`（译注：确实就是DAG的一条边。）`。尽管大部分的 _运算（Operation）_ 或者说Node都只会产生（define）一个值`（译注：可理解为运算的返回值）`，但是实际上`SDNode`也可能会有多个值。比方说有一个Node是`sincos`，它就需要同时返回正弦和余弦值。对这种情况，其他节点的输入就必须要知道具体引用了它的哪一个返回值。此时 _边_ 就需要用`SDValue`而不能是简单的`SDNode`来表示。`SDValue`是一对值`<SDNode, unsigned>`，它不光指明了值的Node，还指明了它使用的是节点的第几个返回值。此外，每个值都是有类型的，所以都会有一个`MVT（Machine Value Type）`来表示它的类型。

> 译注: `SDValue`在LLVM中经常作为`SDNode`的 _Reference_ 来使用；


SelectionDAG有两种类型的值，分别表示 _数据流（Data Flow）_，另外一种则是表示 _控制流的依赖关系（Control Flow Dependencies）_。

> 译注：这里我可能断句有点问题。不过大体上意思没差。
> 所以，又到了举栗子的时间了！简单入门一下数据流和控制流依赖。
> 有这么一段代码（:= 是 define 的意思）

> ```
>	...
>	x := ...	// 把某某表达式定义成x
>	y := ...
>	z := x + y
> ```

> 那么这里的z对x就是数据流上的依赖。因为显然x一遍，z的值就变了。

> 又有这么一段代码
>	if x: z := 3 else z := 4
> 那么这个时候z对x当然也是有依赖的。显然x变了，执行路径可能就变了，z就变化了。这是控制流上的依赖。
> 当然对这个例子，你也可以理解成`z = cond(x, 3, 4)`，进而解释成数据流上的依赖，那当然也能行得通。

如果一个值在数据上依赖于其它Node，值类型就是数据的类型，比方说是个整型或者是浮点。但是如果这个值表示的是一个控制依赖，那么我们有一个更贴切的概念 _链（chain）_ 来命名这一类关系，它在LLVM中的类型是`MVT::Other`。并且，LLVM需要为所有有 _副作用（Side-effects）_ 的Node（例如Loads，Stores，Calls，Returns）提供一个排序。所有有副作用的节点都必须要接受一个 _令牌链（token chain）_ 作为输入，并且产生一个新的链作为输出。LLVM约定，输入的令牌链作为Node的第一个输入参数（operand #0），输出的令牌链作为Node的最后一个输出。

> 译注：Token Chain在实现上就是一个Node Chain。

SelectionDAG有两类特殊的节点，_Entry_ 和 _Root_。_Entry_ 节点使用`ISD::EntryToken`作为 _Opcode_，而 _Root_ 节点是整个 _令牌链_ 中最后一个副作用节点。例如，如果一个函数体只有一个基本块，那`return`这个节点就是 _Root_ 节点。`

> 译注：文档中没有解释这两个节点的作用。多两句嘴。 

> * _Entry_是整个DAG的入口，它标出了一个代码段的起始位置。在很多分析中，它都起到了哨兵节点的作用，很多`SDValue`（例如 _Root_）在初始化的时候都是指向 _Entry_ 节点。
> * _Root_ 作为最后一个有副作用的节点，可以逆行向上索引到所有有副作用（也可以认为是有内存操作）的节点。这样做起别名分析、Scheduling、或者是Auto-Vectorization来就会很方便。此外，对volatile的读写操作，LLVM在构建Chain的时候有一些非常有趣的行为。

最后，SelectionDAG分为 _合法（Legal）_ 和 _非法（illegal）_ 两种。一个 _合法_ 的DAG中，所有节点的指令和参数类型都是目标平台直接支持的。比方说在32bit的PowerPC上，`i1`、`i8`、`i16`、`i64`这些类型就是 _非法_ 类型`（译注：32位PPC上就只有i32可用）`。所以说要翻译成机器直接运行的指令，得将包含了各种复杂数据类型和指令的 _非法_ DAG，通过 _类型合法化（Legalize Types）_ 和 _操作合法化（Legalize Operations）_ 两个阶段转换成一个 _合法_ 的DAG。

<h4>基于SelectionDAG的指令选择过程</h4>
基于SelectionDAG的指令选择是个很麻烦的过程，细分的话有以下几个阶段：

1. 构建最初的DAG —— 首先将LLVM IR`（译注：LLVM IR在内存中也有个保存形式，和SelectionDAG不一样但是很相似）`转化成一个 _非法_ 的SelectionDAG。

2. 优化构建好的SelectionDAG —— 对构建好的SelectionDAG做一些简单优化，尽量让SelectionDAG简单一些，并且把一些平台支持的元指令（meta instructions）（例如 _旋转（Rotates）_、`div/rem`指令）识别出来。这样不仅可以优化最终代码，也可以让指令选择阶段变得稍微简单一些。

3. 合法化SelectionDAG类型 —— 把目标平台不支持的类型统统超度。

4. 优化SelectionDAG —— 合法化阶段肯定得有一些垃圾代码，把它们都消灭。

5. 合法化SelectionDAG的操作（指令） —— 超度目标平台不支持的指令。

6. 优化SelectionDAG —— 再次优化，消除前个阶段带来的一些问题。

7. 指令选择 —— 指令选择子（instruction selector）会根据DAG上的操作（指令）选择合适的平台指令。这一步将会把平台无关的DAG转换成平台相关的DAG。

8. SelectionDAG的调度与队列化（Scheduling and Formation） —— 最后一步是要将DAG重排成指令队列，并且发射到`MachineFunction`中。这一步LLVM使用的传统的Prepass调度算法。

> 译注：解释一下调度算法：

> 指令调度就是给所有的指令排个顺序（这个算法与很多优化都有关系，比方说延迟隐藏、Cache Missing Ratio之类的）。这里Prepass Scheduling是指在寄存器分配前进行指令调度。如果是指令调度在寄存器分配之后，那就成为Postpass Scheduling。

>两者各有利弊。前者的问题在于，指令调度好了之后，寄存器分配时很可能会面临寄存器不够用的情况，那就得加入Spilling Code（寄存溢出代码），用内存来进行数据的周转（和虚拟内存差不多是一个道理）。这部分代码会影响到指令的延迟，这样会非常影响调度的效果。采用Postpass的方法，那就得在寄存器分配完才能进行调度，但此时就没有SSA了，依赖分析上就会差很多，指令调度就会很难做。

在以上所有工作都完成后，SelectionDAG就能被销毁了，代码生成就可以进行接下来的一些工作。

为了满足大家的偷窥欲，LLC提供了一些参数，将编译的中间过程可视化出来：

* `-view-dag-combine1-dags` 可以显示初始构建、还没被优化的DAG。
* `-view-legalize-dags` 可以显示在合法化之前的DAG（已经经过一些优化了）。
* `-view-dag-combine2-dags` 可以显示第二次优化之前的DAG（已经经过一些优化了）。
* `-view-isel-dags` 可以显示指令选择之前的DAG。

如果一切OK的话，在你敲完这些命令之后稍等片刻就会弹出一个窗口把DAG给绘制出来。如果你除了错误提示别的都看不到，那可能是配置出问题了，重新按照文档配置一下LLVM吧`（译注：一般就是什么dot啊，graphviz之类的画图软件配置）`。最后，还有个命令`-view-sunit-dags`可以显示Scheduler的依赖图。这个依赖图是依据最终的SelectionDAG绘制出来。在这张图里面，我们可以知道那些指令 _bundle_ 到了一起。不过它省略了一些与指令调度无关的立即数和节点。

<h4>创建初始的SelectionDAG</h4>
SelectionDAG的创建是个基本的窥孔算法。LLVM IR经过`SelectionDAGBuilder`的处理后转换成SelectionDAG。我们希望这一阶段
可以给SelectionDAG附加尽可能多的底层与平台相关的细节。这个Pass基本都是人肉的。你得人肉的把IR中的`add`指令转换成SelectionDAG中的`add`节点，也要把IR中的`GetElementPtr`指令人肉展开成地址的算术运算。

> 译注：`GetElementPtr`是LLVM IR的指令，用于计算结构体成员或者数组元素的地址偏移量。

根据平台的不同，call，return，varargs这些指令的在转换成DAG的时候都会有所差异，可以使用`TargetLowering`接口作为 _hook_实现平台相关的特性。

<h4>合法化(Legalize)SelectionDAG中的类型</h4>

这一步要把DAG中用到的数据的类型统统转化成被目标平台直接支持的类型。达到这个目的有两种主要途径，一个是将较小的类型转成较大的类型，即 _类型提升（promoting）_，一种是将较大的类型拆分成多个小的类型，即 _展开（Expanding）_。比方说在某些平台上只有64位浮点和32位整数的运算指令，你得把所有f32都 _提升（promoted）_ 到f64，并把i1/i8/i16都提升到i32，同时要把i64拆分成两个i32来存储。当然在提升或扩展的过程中可能会遇到一些问题，例如整型位数变化后，可能需要 _带符号扩展（signed extension）_ 或 _无符号扩展（zero extension）_。无论怎样，最终都要保证最终的指令与原始的IR行为上要完全一致。

LLVM IR中有目标平台无法支持的矢量`（译注： LLVM IR还支持任意长度的 _矢量（vector）_。译注：LLVM中的矢量是为SIMD操作设计的。但是目标平台可能无法支持任意长度的矢量，例如x86上SSE的一些指令只支持4个单精度浮点的矢量。）`，LLVM也会有两种转换方案，_加宽（widening）_，即将大vector拆分成多个可以被平台支持的小vector，不足一个vector的部分补齐成一个vector`（译注：例如在x86平台上将15个f32的vector补齐成16个f32然后拆分成四个m128）`；以及 _标量化（scalarizing）_，即在不支持SIMD指令的平台上，将矢量拆成多个标量进行运算。

在平台特定的`TargetLowering`实现的时候，要用`addRegisterClass`注册平台的寄存器分类及对应的数据类型。 _Legalizer_会根据这些信息来确定平台所支持的类型。

<h4>合法化SelectionDAG 中的操作符</h4>
这一步将DAG中节点的操作/指令合法化成平台支持的操作。

即便数据都是平台所支持的，它也不太可能提供IR中的每一条指令。例如x86中就没有 _条件赋值（conditional moves）_ 指令，PowerPC也不支持从一个16-bit的内存上以符号扩展的方式读取整数。因此，_合法化_ 阶段要将这些不支持的指令按三种方式转换成平台支持的操作： _扩展（Expansion）_，用一组指令来模拟一条操作； _提升（promotion）_ 将数据转换成更大的类型， _定制（Custom）_ 通过Hook，让用户实现合法化。

在平台对应的`TargetLowering`初始化时，需调用`setOperationAction`注册不被支持的操作，并指定三种行为的一种来完成合法化。

如果没有 _合法化_，那么所有的平台上的选择子都需要支持DAG中的每一条操作和所有数据类型。借助于 _合法化_，我们可以让平台间共享全部的规范化模式（cononicalized pattern）。并且因为规范化后的代码仍然是DAG的，因此优化起来也非常方便。`（译注：这里不甚明了cononicalized pattern的具体含义，因此只是直译。不过猜测此处的Pattern应该是指那些平台无关的模式匹配规则。）`

<h4>SelectionDAG的优化</h4>
在整个代码生成阶段，SelectionDAG会被优化多次。在SelectionDAG构建完成之后就需要进行首次优化，完成一些代码的清理工作（例如根据指令参数的具体类型进行的优化）。其他的几轮优化用于清除合法化带来的一些垃圾代码。有了单独的优化pass，合法化就可以不用过于操心生成的代码质量，这样就可以变得简单一些。

另外在指令选择的时候，会生成一大堆的整数带符号扩展或者无符号扩展，这些代码是优化的重点。目前LLVM里面还是 _人肉（ad-hoc）处理_ 它，以后我们会该用更严谨的算法去实现，例如：

“Widening integer arithmetic” 
Kevin Redwine and Norman Ramsey 
International Conference on Compiler Construction (CC) 2004

“Effective sign extension elimination” 
Motohiro Kawahito, Hideaki Komatsu, and Toshio Nakatani 
Proceedings of the ACM SIGPLAN 2002 Conference on Programming Language Design and Implementation.

<h4>选择机器指令</h4>
_选择阶段（Select Phase）_ 是所有平台相关的指令选择过程的统称。这一阶段的输入是合法的SelectionDAG，通过模式匹配出平台指令，并将结果输出到一个新的DAG。考虑一下LLVM IR片段：

> 译注：Legalize的阶段和Select阶段都会变换SelectionDAG中的Operation。只不过前者是将目标无关的Operation转换成目标无关但是会被平台支持的Operation，后者是将平台支持的Operation变成平台的指令。

```
%1 = fadd float %w, %x 
%2 = fmul float %1, %y 
%3 = fadd float %2, %z
```

这段代码对应的SelectionDAG大概是下面这样：

```
（fadd:f32 (fmul:f32 (fadd:f32 W, X), Y), Z)
```

如果目标平台支持浮点的 _乘加（multiply-and-add，FMA）_ 操作，那么此时会把add和mul指令合并到一起。在PowerPC上，输出的DAG是这样：

```
(FMADDS (FADDS W, X), Y, Z)
```

`FMADDS`是一个单精度的三元运算符，它将头两个参数相乘并加到第三个参数上。`FADDS`是一个单精度的二元加法运算符。为了执行这个模式匹配，PowerPC的后端得有这样的指令定义：

```
def FMADDS
	: AForm_1<59, 29,
     (ops F4RC:$FRT, F4RC:$FRA, F4RC:$FRC, F4RC:$FRB),
      "fmadds $FRT, $FRA, $FRC, $FRB",
      *[(set F4RC:$FRT, (fadd (fmul F4RC:$FRA, F4RC:$FRC),
                                           F4RC:$FRB))]*>;
def FADDS
	: AForm_2<59, 21,
     (ops F4RC:$FRT, F4RC:$FRA, F4RC:$FRB),
      "fadds $FRT, $FRA, $FRB",
      *[(set F4RC:$FRT, (fadd F4RC:$FRA, F4RC:$FRB))]*>;
```

加粗的部分就是对模式匹配的提示。所有的DAG操作/指令例如`fmul/fadd`都在`include/llvm/Target/TargetSelectionDAG.td`。`F4RC`是输出和输出的寄存器分类。

TableGen会读入`.td`文件，并且生成对应的模式匹配代码。这样做有下面好处：

* 编译时间`（译注：此处是指编译编译器的时间。compiler-compiler time）`内可以检查所有的指令模式并且告诉你模式是不是正确的。
* 模式匹配时能处理对参数的任意 _约束（constraints）_。你可以很容易的描述这样的约束：“参数必须是任何13bit、带符号扩展的立即数”。作为例子，你可以参考PowerPC后端中的`immSExt16`和相应的`tblgen`类。
* 能够识别一些重要的pattern。例如它知道加法是符合交换律的，因此它既能匹配`fadd X, (fmul Y, Z)`，也能匹配`fadd (fmul X, Y), Z`。这些都不需要用户做任何特别的处理。
* 它有完备的类型推导能力（type-inferencing system）。大部分情况下用户可以不用显式的指定模式参数类型。`FMADDS`就不需要在tblgen中指明这个pattern是f32的。它会根据参数描述中`F4RC`这个寄存器分类推导出它只能匹配f32的数据。
* 目标平台可以定义自己的" _模式片段（pattern fragments）_"，这是一组可复用的模式的集合。例如SelectionDAG不支持`not`操作，但是用户却能定义一个`not x`的 _模式片段_，它可以展开成`xor x, -1`。可以参见片段`not`和`ineg`的相关代码。
* 除了指令之外，目标平台还可以通过类`Pat`指定将任意模式匹配成指令。考虑PowerPC上，无法在一条指令内，读取一个任意整型立即数到寄存器中。那么可以在`tblgen`中添加这样的pattern:

```
// Arbitrary immediate support.  Implement in terms of LIS/ORI.
def : Pat<(i32 imm:$imm),
          (ORI (LIS (HI16 imm:$imm)), (LO16 imm:$imm))>;
```

如果平台不能没有单指令读入立即数到寄存器的pattern，它就会用这条规则：“将一个i32立即数的读取转化成一条ORI（同16bit立即数进行或操作）和一条LIS（左移十六位）指令”。在匹配规则后，输入的立即数也会拆分成`LO16/HI16`以匹配这条规则。
* 系统在大多数时候是自动匹配的，但是也允许你添加自定义的C++代码去处理一些非常复杂的匹配规则。

当然，它也存在着不少缺陷。当然，大部分问题将在以后的开发中逐渐的完善。

* 暂时还没有办法匹配有多个值的SelectionDAG节点，例如`SMUL_LOHI`, `LOAD`, `CALL`。这也是用户需要写C++代码的主要原因。
* 对于复杂的寻址模式目前还支持的不够好。（例如x86的四参数寻址模式，目前LLVM还是用C++代码来匹配）。以后我们会增加一些模式片段来支持这一特性。我们还有打算扩展模式片段使得一个模式片段可以支持多个不同的模式。
* 不能自动推断标记`isStore/isLoad`。
* 不能自动产生Legalizer需要的寄存器支持集和指令支持集。
* 没有办法和自定义的合法化节点协同工作。（dont have a way of tying in custom legalized nodes yet.）

> 译注: 两个问题有待考证： 1. 不清楚isStore和isLoad的标记（flag）有什么作用；2. 不知道自定义的合法化节点是什么样子的节点。

尽管有这些问题，指令选择子产生器仍然是非常有用的，对大部分平台，它都能涵盖绝大部分的二元和逻辑指令。如果你遇到了问题或者有什么不懂的，可以与Christ联系！`（吐槽：邮件列表里面都是小弟在回。而且口气都很可怕。）`

<h4>队列化SelectionDAG</h4>
调度阶段主要将指令从DAG中提取出来并加以排序。调度器会根据平台的特性，并依据一定的规则（如最小化寄存器压力或者隐藏指令延迟）来对指令排序。在这一步完成后，DAG会被转换成`MachineInstr`的列表，SelectionDAG会随之删除。

尽管这一步在逻辑上是与指令选择分离的，但是它也是在SelectionDAG进行调度和排序，因此它和指令选择阶段的关系仍然是非常紧密的。

<h4>SelectionDAG的工作计划</h4>
1. Optional function-at-a-time selection.
2. Auto-generate entire selector from .td file.

<h3>基于SSA的机器码优化</h3>
TO BE WRITTEN

<h3>变量（值）生存期分析</h3>
变量的生存期是指某个变量的有效范围。变量生存期分析最重要的用途是在程序的同一个点上，是否有一个或多个虚拟寄存器申请了相同寄存器。如果发生了 _冲突（conflit）_ 那么就需要这些虚拟寄存器通过寄存器 _溢出（spilled）_ 的形式共享物理寄存器。

> 译注：这里所说的变量（variable）通常是指值（value）。

<h4>活动变量分析</h4>
活动变量分析的第一步是清除那些定义了但是从未引用过的寄存。其次是确定最后一条引用某个虚拟寄存器指令。所有的虚拟寄存器以及参与分配的物理寄存器都需要进行活动变量分析。因为LLVM是SSA的形式，所以很容易分析虚拟寄存器的生存期`（译注：因为寄存器不能被写，所以生命期就很短）`。并且因为LLVM约定了物理寄存器不能跨基本块，物理寄存器的生命期只需在块内分析即可。不参与分配的寄存器，如栈指针和条件寄存器，是不跟踪其生命期的。

如果考虑到函数调用和返回（live in to or out of a function），那么物理寄存器的生命期可能会跨越函数边界。LLVM通过 _虚拟的（dummy）_ `defining`指令保留了所有 _输入寄存（live in register）_ 的生存期；如果最后一个基本块拥有`return`指令，那么函数所有的 _返回值（live out of register）_ 的生存期皆用此条指令来界定。

`PHI`节点需要做一些特殊处理。变量生存期分析是对函数控制流图（CFG, Control Flow Graph）的深度优先遍历。这样在分析`PHI`节点时，它的参数寄存器可能还未定义。所以在访问到`PHI`节点的时候，仅处理`PHI`节点的定义，而它所引用的寄存器则在其他块中处理。

对于当前块的每个`PHI`节点（Phi node of current basic block），我们都在块末模拟一条赋值指令（simulate an assignment at the end）并遍历所有的后继块。如果后继块（successor basic block）中存在`PHI`节点并且引用的参数来自于当前块，那么会从当前块向其所有 _前驱块（predecessor basic block）_ 追溯，直到找到定义该变量的指令，并认为该变量在这一范围内都是 _存活的（alive）_。

> 译注：这段文字大部分来自注释，但是它又没有引用完整，因此很难理解。
> 考虑以下的代码：

> ```
> Block0:
>   ...
>   %In0 = ...
>   ...
> Block1:
>   %In2 = phi [%In0, Block1], [%In1, Block2]
>   ...
> Block2:
>   ...
>   %In1 = ...
>   ...
>   ... = add %In2, ...
> ```

> 可以看到，

> 1. 在读到phi指令的时候，In0已经定义了，但是In1并没有定义。
> 2. Phi的节点在Block2之前。如果将Phi节点作为In1生存期的终结，它显然还没开始就结束了。

> 因此，在处理Phi节点及其引用的寄存器的生存期，需要一些技巧。具体的，

> 1. 在生存期分析之前，要收集Phi参数的来源块。（这个很重要！）
> 2. 将Phi节点挂接到来源块上。（所谓当前块的Phi节点，并不是指当前块内的Phi节点，而是指挂接在块上Phi节点）
> 3. 在块中的其他指令节点处理完毕后，处理它所关联的Phi节点
> 4. 对于当前块所关联的Phi节点的参数，我们认为它在当前块和当前块的前驱块中均是alive的（所谓虚拟的assignment），这一范围一直追溯到参数寄存器定义的地方。
> 5. 因为Phi节点的参数寄存器的生命期都是在它的来源块中处理掉的，所以在Phi节点出现的地方只需要处理它所定义的寄存器即可。

> 此外，在存储和表达变量生存期的时候，代码和文档也略有区别，具体内容可以参见`include/llvm/CodeGen/LiveVariables.h`

<h4>生存期分析（Live Intervals Analysis）</h4>

在获得了活动变量之后就可以进行进一步的分析。在此之前，要给所有的块和机器指令进行编号；紧接着处理 _输入值（“Live-in” values）_，它们都是物理寄存器，因此会在基本块的末尾结束生存期（killed）；如果将机器指令表示成`[1, N]`的序列，那么虚拟寄存器的生存周期可以表示为`[i, j)`，其中 `1 <= i <= j < N`。

（More to come ...）

<h3>寄存器分配</h3>
_寄存器分配（Register Allocation）_ 是把程序从无限的虚拟寄存器机模型映射到有限的物理寄存器机模型的必要阶段。不同的平台可以使用的物理寄存器数量是不同的。通常来说，平台所提供的物理寄存器，都要远小于虚拟寄存器数量，此时需要把一些虚拟寄存器映射到内存中。这些在内存中的虚拟寄存器，我们称之为 _溢出寄存器（Spilled Virtuals）_ 。

<h4>寄存器在LLVM中的表达</h4>
每一个物理寄存器在LLVM中均有一个 1 - 1023 范围内的编号。具体的编号细节，可以在`GenRegisterNames.inc`文件中找到。例如在`lib/Target/X86/X86GenRegisterInfo.inc`中可以看到32bit的EAX寄存的编号是43，MMX寄存器`MM0`编号是65。

在一些架构上，不同的寄存器名可能会共享相同的物理空间。x86中`EAX`，`AX`，`AL`就共享了8个bit。这些寄存器在LLVM中统称为 _别名（aliased）_。别名信息可以在平台的`RegisterInfo.td`中查到。通过`MCRegAliasIterator`，可以遍历所有的寄存器别名。

LLVM将全部物理寄存器按照 _寄存器分类（Register Classes）_ 进行分组。每一个寄存器分类中的所有寄存器，在功能上都是等价的，彼此间可以相互替代。虚拟寄存器也有分类，而且它只能被映射到同分类的物理寄存器上。还是以x86为例，如果虚拟寄存器是8bit的，那么它也只能映射到8bit的物理寄存器上。在代码中寄存器分类是在`TargetRegisterClass`的实例中描述的。LLVM通过以下代码，判断虚拟寄存器和物理寄存器是否兼容：

```C++
bool RegMapping_Fer::compatible_class(MachineFunction &mf,
                                      unsigned v_reg,
                                      unsigned p_reg) {
  assert(TargetRegisterInfo::isPhysicalRegister(p_reg) &&
         "Target register must be physical");
  const TargetRegisterClass *trc = mf.getRegInfo().getRegClass(v_reg);
  return trc->contains(p_reg);
}
```

在某些情况下，特别是调试的时候，可能需要修改可用的物理寄存器。此时可以通过修改`TargetRegsterInfo.td`。用`grep`在文件中找一下`RegisterClass`，可以看到它最后的参数是一个寄存器列表。只需要修改这些列表，就可以排除不需要的寄存器。另外，也可以显式的将寄存器排除在 _分配顺序（Allocation Order）_ 外。作为用例，你可以参考`lib/Target/X86/X86RegisterInfo.td`中的寄存器分类`GR8`。

虚拟寄存器也有相应的编号。和物理寄存器不同，虚拟寄存器是没有别名的。此外，虚拟寄存器不是在`TargetRgisterInfo.td`中预定义，而是通过`MachineRegisterInfo::createVirtualRegister()`运行时分配出新的虚拟寄存器来。`IndexedMap<Foo, VirtReg2IndexFunctor>`保存了虚拟寄存器的信息。如果想遍历全部的寄存器，可以参考以下代码：

```C++
for (unsigned i = 0, e = MRI->getNumVirtRegs(); i != e; ++i) {
  unsigned VirtReg = TargetRegisterInfo::index2VirtReg(i);
  stuff(VirtReg);
}
```

在寄存器分配之前除了，少数必须要用物理寄存器的情况，基本上指令的参数都使用虚拟寄存器。通过一些函数可以了解寄存器的使用情况：`MachineOperand::isRegister()`判断指令参数是不是用一个寄存器；`MachineOperand::getReg()`获得参数对应的寄存器号。此外，寄存器既可以是指令的 _输入（use）_，也可以是指令的 _输出（def）_，比方说	 `ADD reg:1026 := reg:1025 reg:1024` 就相当于 `reg:1026 = reg:1025 + reg:1024`。输入输出的信息，可以通过`MachineOperand::isUse()`和`MachineOperand::isDef()`获得。

在寄存器分配之前就已经占用的物理寄存器，我们称为_预着色寄存器（pre-colored registers）_。这些寄存器之前我们提到过，主要用在函数参数的寄存器传递、返回值、以及一些特殊的指令上。预着色寄存器可能是 _隐式定义（implicitly defined）_ 的，也可能是 _显式定义（explicitly defined）_的。显式定义的寄存器就是指令的参数，可以通过`MachineInstr::getOperand(int)::getReg()`来获取。隐式定义的寄存器，则需要通过`TargetInstrInfo::get(opcode)::ImplicitDefs`来获得。隐式和显式的预着色寄存器定义主要区别在前者是在定义指令的时候静态指定的，而后者会根据程序而有所变化。举个例子，`call`指令就是典型的隐式寄存器定义，因为它始终占用同样的寄存器。通过`TargetInstrInfo::get(opcode)::ImplicitUses`，可以查找指令隐式的定义了哪些物理寄存。注意，不管用什么寄存器分配算法，预着色的寄存器都是要强制满足的。只要预着色的寄存器还在使用中（alive），它们就不能被虚拟寄存器的值所覆盖。

<h4>虚拟寄存器到物理寄存器的映射</h4>
实现虚拟寄存到物理寄存或 _内存插槽（memory slot）_的映射 有两种方案，_直接映射（direct mapping）_ 和 _间接映射（indirect mapping）_。前者使用`TargetRegisterInfo`和`MachineOperand`中的API即可，后者则需要`VirtRegMap`以正确的插入读写指令实现内存调度。

对寄存器分配器算法的开发者来说，直接映射要更加灵活。但是它的实现相对复杂，也比较容易出错。用户需要自行插入内存的操作指令，以处理虚拟寄存器在内存中的映射。开发者需要用到的API有: `MachineOperand::setReg(physicalReg)`建立虚拟寄存器和物理寄存器之间的映射；`TargetInstrInfo::storeRegToStackSlot(...)`将虚拟寄存器保存到内存中；`TargetInstrInfo::loadRegFromStackSlot`从内存中读出虚拟内存的值。

间接映射的API可以帮助开发者从内存存取的细节中解脱出来。用户只需要通过`VirtRegMap::assginVirt2Phys(virtualReg, physicalReg)`建立物理寄存器和虚拟寄存器的映射，或者将`VirtRegMap::assignVirt2StackSlot(virtualReg)`将虚拟寄存器映射到内存中即可。你还可以通过`VirtRegMap::assignVirt2RegSlot(virtualReg, stackLocation)`强制将虚拟寄存器映射到某个内存地址上。

不过有一点需要强调，即便将虚拟寄存器映射到内存上，实际指令在执行的时候，往往还是要先读到物理寄存器当中的`（译注：特别是一些RISC架构的CPU）`。这个时候我们可以假设，LLVM是先把物理寄存器的值保存下来，然后读入虚拟寄存器的值，执行指令，再把虚拟寄存器的值保存好了，最后恢复物理寄存器的现场。`（译注：这个时候寄存器分配算法如果比较好的话，会有空余的物理寄存器，就没必要这么纠结了。）`

如果用户使用了LLVM提供的间接映射将虚拟寄存器映射到物理寄存器或内存上，那么就需要 _溢出代码（spiller object）_ 插入一些内存读写指令。所有映射到内存上的虚拟寄存器都会在定义的时候把值写到内存中，到引用的时候再读出来。溢出代码在实现的时候是经过优化的，以避免不必要的读写。`lib/CodeGen/RegAllocLinearScan.cpp`中的`RegAllocLinearScan::runOnMachineFunction`演示了溢出代码的典型用法。

<h4>处理双参数的指令</h4>
除了少数Call这样的指令外，LLVM大部分的机器指令都是三参数的（通常是两个参数一个返回值）。但是大部分的硬件指令都是双参数。其中一个参数既是输入，同时也是输出。例如x86中的`ADD %EAX, %EBX`实际上等价于`%EAX = %EAX + %EBX`。

所以LLVM在寄存器分配前，有一个独立的Pass`TwoAddressInstructionPass`将三参数指令转换成双参数的形式。例如`%a = ADD %b %c`会被转换成下面的代码：

```
%a = MOV %b 
%a = ADD %a %c 
```

从例子中可以知道，经过双参数转换后的指令就不再是SSA的形式了。此外还要注意的是，第二条指令在实现中实际表达为`ADD %a[def/use] %c`，说明参数`%a`既是输入，也是输出。

<h4>解构SSA</h4>
在寄存器分配阶段，需要将SSA形式的代码进行 _解构（SSA Deconstruction Phase）_。SSA是个好东西，分析方便，优化方便。但是硬件没办法直接执行它，最起码一个Phi节点/指令，就让硬件嗝屁了。所以要让SSA执行起来，首当其冲就是既要维持原有代码的行为，又要清理掉其中的Phi节点。

清理Phi指令的办法有很多。最传统的、也是LLVM使用的方法，就是用 _拷贝（Copy）指令_ 替换Phi指令。具体实现可以参阅`lib/CodeGen/PHIElimination.cpp`。不过在清理Phi指令之前，先要为每一条Phi指令都分配一个PHIEliminationID。

<h4>指令折叠</h4>
_指令折叠（Instruction Folding）_ 是一个优化，目的是移除多余的拷贝指令。例如下段代码

```
%EBX = LOAD %mem_address
%EAX = COPY %EBX
```
可以被安全的优化成

```
%EAX = LOAD %mem_address
```

`TargetRegisterInfo::foldMemoryOperand(...)`提供了具体的优化算法。在执行指令折叠的时候必须要小心谨慎，折叠后的指令有可能与原先指令完全不同。`lib/CodeGen/LiveIntervalAnalysis.cpp`中的`LiveIntervals::addIntervalsForSpills`是指令折叠的一个范例。

<h4>LLVM自带的寄存器分配算法</h4>
LLVM为应用程序开发者提供了以下几种寄存器分配算法：

* _Fast_ —— Debug版本中默认的寄存器分配算法。它在基本块层面上处理寄存器分配，尽可能保留寄存器的值，并且在适当的时候才复用寄存器。
* _Basic_ —— Basic是增量的寄存器分配算法。它通过一个启发式（Heuristics）算法按一定顺序分配给寄存器生存期。因为这一算法可以在运行时进行调整，因此它可以允许以扩展的形式开发一些非常有趣的寄存器分配方案。虽然这个算法作为产品级算法还不够称职，但是它可以用来作为正确性和性能的评价基准。
* _Greedy_ —— 贪心（Greedy）算法是LLVM默认的寄存器分配算法。它可以看成是Basic算法将变量生存期进行分裂（splitting global live range）后高度优化的版本。贪心算法大大减少了 _溢出代码_ 带来的成本。
* _PBQP_ —— 这个看起来很专业的名字来自于 _Partitional Boolean Quadratic Programming_ 的缩写。它将寄存器分配描述为一个分区布尔二次规划的问题，解算PBQP后，将结果用于寄存器分配。

> 译注：

> 1. PBQP是规划算法的一类，相关资料可以参见 http://www.complang.tuwien.ac.at/scholz/pbqp.html
> 2. 关于PBQP在寄存器分配中的应用，可以参见此篇 http://pp.info.uni-karlsruhe.de/uploads/publikationen/buchwald11cc.pdf
> 3. 其实寄存器规划是个大问题。教科书上通常会将寄存器分配转化为相交图（Interference Graph）上的节点着色算法。K着色问题本身是NP完全问题，而且因为物理寄存器数量较少，经常需要溢出到内存。最小化溢出代价也是个NP完全问题。
> 4. Basic算法根据实现，应接近Poletto和Sarkar提出的线性扫描算法（Linear Scan）。它实际上是图着色的简化。
> 5. 最后，关于SSA的寄存器分配问题，在2005年证明了SSA表达下的相交图是弦图（Chordal Graph）。弦图有个非常重要的性质，就是它可以在多项式时间内着色。

通过llc的参数，可以指定不同的寄存器分配算法：

```sh
$ llc -regalloc=basic file.bc -o ln.s
$ llc -regalloc=fast file.bc -o fa.s
$ llc -regalloc=pbqp file.bc -o pbqp.s
```

<h3>Prolog/Epilog的生成</h3>

<h4>压缩的Unwind</h4>
`（译注：这段文档好像没有写完，很混乱。Prolog和Epilog是进入和退出函数时执行的辅助代码。本节主要讨论Prolog和Epilog对异常的支持。）`

如果函数体内需要处理异常，那么在退出函数的时候，需要执行一段叫Unwinding的代码完成清理。Unwinding需要一些栈帧信息，其中一种常见的信息格式成为DWARF。但是DWARF最初是设计给调试器的，每个函数的 _帧描述（Frame Description Entry，FDE）_ 都有20-30个字节之多。并且它在运行时需要根据函数内的地址查找FDE，这也是一比不小的开销。因此LLVM使用了一种称为Compact Unwind的格式将每个函数的开销降低到四个字节。

Compact Unwind是一个32位的编码，根据平台会有不同的编码方式。它保存了在异常发生后需要恢复的寄存器和恢复数据的来源，以及如何Unwind出函数。链接器（Linker）在创建最终可执行文件的时候，会增加`__TEXT`和`__unwind_info`两个段（section)。任何函数，只要有Compact Unwind，Unwind数据都会被编码到这两个段中。两个段都很小，而且在运行时通过函数查找Unwind数据的速度也很快。如果是完整的DAWRF Unwind信息，那么所有的FDE都被保存在`__TEXT`和`__eh_frame`段中，`__TEXT`和`__unwind_info`只保存FDE的偏移。

对于x86来说，有三种Compact Unwind编码模式：

1. **具有栈帧指针的函数（`EBP`/`RBP`）**。有`xBP`指针的函数，在调用的时候会在返回地址之后将栈顶指针压栈，然后再将`xSP`赋值给`xBP`。因此在Unwind的时候，要将`xSP`恢复成`xBP`并将`xBP`从栈中弹出。此外，还要从`xBP-4`到`xBP-1020`的堆栈范围内恢复所有的 _非易失性存储器（non-volatile registers）_。其偏移量在32位上除以4，或64位上除以8后，编码到16-23bit中（对应掩码是`0x00FF0000`）。需要保存的寄存器按照每3bit表示一个寄存器号的形式编码到0-14bit（对应掩码是0x00007FFF）。下表为寄存器号的对照表：

<table class="table table-bordered table-striped table-condensed">
   <tr>
      <th>Compact Number</th>
      <th>i386 Register</th>
      <th>x86-64 Register</th>
   </tr>
   <tr>
      <td>1</td>
      <td>EBX</td>
      <td>RBX</td>
   </tr>
   <tr>
      <td>2</td>
      <td>ECX</td>
      <td>R12</td>
   </tr>
   <tr>
      <td>3</td>
      <td>EDX</td>
      <td>R13</td>
   </tr>
   <tr>
      <td>4</td>
      <td>EDI</td>
      <td>R14</td>
   </tr>
   <tr>
      <td>5</td>
      <td>ESI</td>
      <td>R15</td>
   </tr>
   <tr>
      <td>6</td>
      <td>EBP</td>
      <td>RBP</td>
   </tr>
</table>

2. **无栈帧，函数的堆栈大小固定且较小的（不使用`xBP`保存堆栈指针）**。此种情况下在函数返回时直接向`xSP`加上一个地址偏移。此时所有的非易失性寄存器都保存在返回地址之后。堆栈大小（仍然是除以4或除以8）编码到16-23bit，因此这一模式仅支持1024字节（32bit）或2048字节（64bit）以内的堆栈大小；被保存的寄存器数量编码到9-12bit（掩码为`0x00001C00`）；而0-9bit（掩码为`0x000003FF`）则按顺序编码了被保存的寄存器号。（具体的编码算法可以参见`lib/Target/X86FrameLowering.cpp`中的`encodeCompactUnwindRegistersWithoutFrame()`）。

3. **无栈帧，函数的堆栈大小固定且较大（不使用`xBP`保存堆栈指针）**。此种情况与上一种情形类似，但因为堆栈太大因而无法将堆栈大小编码到Unwind中，因此选择在Prolog中插入指令`subl $nnnnn, %esp`以调整堆栈。Compact Unwind则将`$nnnnn`的偏移量保存在9-12bit中（掩码0x00001C00）。

<h3>机器码的最终优化</h3>
TO BE WRITTEN

<h3>Code Emission</h3>
简单来说，_代码发射_ 就是将 _MachineFunction_ 和 _MachineInstr_ 等通过 MC Layer 的API（`MCInst`，`MCStreamer`等）输出出去。它可能由以下几个类来完成：平台无关的`AsmPrinter`，平台相关的`AsmPrinter`的子类（如`SparcAsmPrinter`）,以及`TargetLoweringObjectFile`。

因为MC Layer工作在Object file一级上，因此它已经没有了函数、全局变量的概念。在MC Layer中，你需要和 _标签（labels）_、 _指示字（directives）_ 和 _指令（instructions)_ 打交道。前文也提到过，MC Layer中最终要的API类是`MCStreamer`。因此只要选择不同的实现，使用`MCStreamer`提供的接口如`EmitLabel`来发射每一条指示字就可以了。

如果你需要为你的平台开发代码生成器，那么以下三个部分是你必须要实现的：

1. 你需要为你的平台提供一个`AsmPrinter`的子类。这个类需要将`MachineFunction`转化成对MC Label的构造。`AsmPrinter`其实已经提供了大多数可以复用的代码，你只需要重写一部分即可。如果你需要为你的平台实现ELF，COFF或者MachO这些格式，那也是非常容易的。你可以从`TargetLoweringObjectFile`中复用大量的公共逻辑。
2. 为平台实现指令打印的功能（instruction printer）。指令打印接受`MCInst`，并以文本形式输出到raw_ostream中。大部分逻辑可以在`.td`中直接定义出来，比方说像`add $dst, $src1, $src2`这样，但是你仍然需要去实现参数（operands）打印的部分。
3. 你还需要将`MachineInstr`转化成`MCInst`。这一过程一般在`<target>MCInstLower.cpp`文件中实现。向底层转化的过程通常是平台相关的，而且也需要将跳转表、常量池索引、全局变量地址等这些上层的概念统统转化成对应的`MCLable`。这一步也需要将一些伪指令（pseudo ops）用真正的机器指令替代。最终生成的`MCInst`，就可以用来编码或打印成文本形式的汇编。

如果你想直接支持`.o`文件的输出，或者实现自己的汇编器，你也可以选择实现一个`MCCodeEmitter`。它的任务是将`MCInst`转化成字节流（code bytes）并进行定位（relocations）。

<h3>用于VLIW架构的指令打包器(Packetizer)</h3>
<h4>将指令映射成功能单元</h4>
<h4>生成并使用Packetization Tables</h4>
<h2>Native Assembler的实现</h2>
<h3>指令解析(Parsing)</h3>
<h3>处理指令别名(Instruction Alias)</h3>
<h4>助记符别名</h4>
<h4>指令别名</h4>
<h3>指令匹配(Matching)</h3>

<h2>特定平台的一些注意事项</h2>
`译注：这一块随着版本变化一直在变，而且也没什么难理解的部分，就容许我偷个懒，不做翻译啦。`

<h3>平台特性矩阵</h3>
<h3>尾调用(tail call)的优化</h3>
<h3>相邻调用(sibling call)的优化</h3>
<h3>x86后端</h3>
<h3>PowerPC后端</h3>
<h3>PTX后端</h3>


