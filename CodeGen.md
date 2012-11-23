#LLVM中目标无关(Target-Indepenent)的代码生成(Code Generator)

* [译序](#prolugue)
* [介绍](#introduction)
* [用于目标描述的类](#tardesc_classes)
* [用于机器码描述的类](#mcdesc_classes)
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

做一个编译器有很多路子。解释器也好，一遍编译也好，都算是编译器的Framework。但是随着语言机制的愈发复杂以及后端技术上的日益完善，编译器中用于语法分析和语义处理的前端和用于生成目标代码的后端的界限也越来越清晰。某种意义上，借助于中间语言（Intermediate Language，IL）或者称中间表达（Intermediate Presentation）的方法已经成为产品级编译器开发中绝对主流的技术。

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

<h2 id = "introduction">介绍</h2>

<h3>代码生成的必要组件			 </h3>
<h3>代码生成器的高层设计			 </h3>
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


