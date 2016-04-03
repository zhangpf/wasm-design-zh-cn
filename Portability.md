# 可移植性

WebAssembly的[二进制格式](BinaryEncoding.md)被设计成可以在多种操作系统和指令集体系结构，以及[Web环境](Web.md)和[非Web环境](Web.md)下高效运行。

## 高效执行的（环境）假设

[受限，局部和非确定性](Nondeterminism.md)的执行环境无法提供下面列出的特性，但是仍然可以运行WebAssembly模块。尽管这些环境不得不模拟主机硬件或操作系统不提供的行为，使得WebAssembly模块看起来*好像*环境支持它们顺利执行。这种方式有时会导致性能低下。

随着WebAssembly标准不断深入，我们期望能下面这些要求，和使WebAssembly适应那些在最初设计时并不存在的新平台的方式能够形式化。

具体来讲，WebAssembly的可移植性需假定执行环境提供如下的特征：

* 一个字节（byte）有8比特（bit）。
* 支持以1个字节为粒度进行访存。
* 支持非对齐内存访问，或者通过可靠的陷入方式以软件模拟来实现。
* 二进制补码的带符号32位整型，以及可选的64位整型。
* IEEE754-2008规范的32位和64位浮点数，但不包括[一些例外](AstSemantics.md#floating-point-operators)。
* 小端序（Little-endian）。
* 能被32位指针和索引高效访问的内存区域。
* wasm64（64位wasm）额外需要支持64位指针和索引访问[超过4GB](FutureFeatures.md#linear-memory-bigger-than-4-gib)的线性内存空间。
* 强制WebAssembly模块与同一主机上其他进程和模块（包括WebAssembly模块和非WebAssembly模块）之间进行安全隔离。
* 执行环境必须为所有线程(即使在非并行方式中)，提供forward progress保证*（译者注：[forward progress guarantees](https://msdn.microsoft.com/en-us/library/windows/hardware/ff543244(v=vs.85).aspx)是在低内存场景下防止丢失关键数据的一种技术）*。
* 当8位，16位或32位自然对齐时，必须支持lock-free原子内存操作，实现该支持的最小代价是具有原子compare-and-exchange操作（或等价的load-linked/store-conditional操作）。
* wasm64额外需要支持64位自然对齐时的lock-free原子内存操作。

## API

WebAssembly没有对任何API或系统调用作规范, 并且已规范的[导入（import）机制](Modules.md)中涉及的具体可用导入项集合也是在主机环境定义的。在[Web](Web.md)环境中， 开发者通过[开放Web平台](https://en.wikipedia.org/wiki/Open_Web_Platform)定义的Web API访问来功能，而[非Web](NonWeb.md)环境可以选择实现标准Web API、标准非Web API（如POSIX）或实现一套自己的接口等方式自由实现。

## 源码层

在C/C++源码层上的可移植性可以通过在标准API（如POSIX接口）上编程，同时依靠编译器和/或库在编译时（通过`#ifdef`）或运行时（通过[特征检测](FeatureTest.md)和动态[加载](Modules.md)/[链接](DynamicLinking.md)），将标准接口映射到主机环境的导入项的方式实现。
