#LLVM中目标无关(Target-Indepenent)的代码生成(Code Generator)

* [译序](#prolugue)
* [介绍](#introduction)
* [用于描述平台的类](#tardesc_classes)
* [用于描述机器码的类](#mcdesc_classes)
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

1. **指令选择（Instruction Selection）** —
2. **调度与格式化（Scheduling and Formation）** —
3. **基于SSA的优化** —
4. **寄存器分配** —
5. **Prolog/Epilog的生成** —
6. **机器码的晚期优化** —
7. **发射代码（Code Emission）** —

<h3>使用TableGen生成目标平台描述	 </h3>
<h2 id = "tardesc_classes">用于描述目标的类</h2>
<h3>class <code>TargetMachine		</code></h3>
<h3>class <code>DataLayout			</code></h3>
<h3>class <code>TargetLowering		</code></h3>
<h3>class <code>TargetRegisterInfo	</code></h3>
<h3>class <code>TargetInstrInfo		</code></h3>
<h3>class <code>TargetFrameInfo		</code></h3>
<h3>class <code>TargetJITInfo		</code></h3>
<h2 id = "mcdesc_classes">用于描述机器码的类</h2>
<h3>class MachineInstr</h3>
<h4><code>MachineInstrBuilder.h</code>中的函数</h4>
<h4>固定（预分配）的寄存器</h4>
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


