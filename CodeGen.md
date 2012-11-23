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
2. **调度与格式化（Scheduling and Formation）** — 耳熟能详的指令重排就是这个部分的主要工作。这个阶段将会读取DAG，并根据需要进行重新排列，最后将指令以`MachineInstr`s输出。不过虽然我们这里是单独列为一个阶段，但是实际上它和 _指令选择_ 都操作的是 _SelectionDAG_，并且两者的实现也极为接近，所以在讲算法的时候，将它和 _指令选择_ 放在一起讨论。
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
<h4>call-clobbered寄存器</h4>
<h4>SSA Form的机器码</h4>
<h3>class <code>MachineBasicBlock</code></h3>
<h3>class <code>MachineFunction</code></h3>
<h3><code>MachineInstr Bundles</code></h3>
<h2 id="mclayer">The 'MC' Layer</h2>
<h3><code>MCStreamer</code> API</h3>
<h3>class <code>MCContext</code></h3>
<h3>class <code>MCSymbol</code></h3>
<h3>class <code>MCSection</code></h3>
<h3>class <code>MCInst</code></h3>
<h2 id="gcalgo">目标无关的代码生成算法</h2>
<h3>指令选择(Instruction Selection)</h3>
<h4>SelectionDAGs简介</h4>
<h4>基于SelectionDAG的指令选择流程</h4>
<h4>创建初始的SelectionDAG</h4>
<h4>合法化(Legalize)SelectionDAG中的类型</h4>
<h4>合法化SelectionDAG 中的操作符</h4>
<h4>优化SelectionDAG</h4>
<h4>选择机器指令</h4>
<h4>调整与格式化SelectionDAG</h4>
<h3>基于SSA的机器码优化</h3>
<h3>变量（值）的生存期分析</h3>
<h4>活动变量分析</h4>
<h4>生存期分析<h4>
<h3>寄存器分配</h3>
<h4>寄存器在LLVM中的表达</h4>
<h4>虚拟寄存器到物理寄存器的映射<h4>
<h4>处理双参数的指令</h4>
<h4>解构SSA</h4>
<h4>指令折叠</h4>
<h4>LLVM自带的寄存器分配算法</h4>
<h3>Prolog/Epilog的生成</h3>
<h3>机器码的最终优化</h3>
<h3>Code Emission</h3>
<h3>用于VLIW架构的指令打包器(Packetizer)</h3>
<h4>将指令映射成功能单元</h4>
<h4>生成并使用Packetization Tables</h4>
<h2>Native Assembler的实现</h2>
<h3>指令解析(Parsing)</h3>
<h3>处理指令别名(Instruction Alias)</h3>
<h4>助记符别名</h4>
<h4>指令别名</h4>
<h3>指令匹配(Matching)</h3>
<h2>特定平台的一些注意事项</h4>
<h3>平台特性矩阵</h3>
<h3>尾调用(tail call)的优化</h3>
<h3>相邻调用(sibling call)的优化</h3>
<h3>x86后端</h3>
<h3>PowerPC后端</h3>
<h3>PTX后端</h3>


