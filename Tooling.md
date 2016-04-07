#  支持工具

在浏览器内执行的工具，（与原生工具相比，）常常具有不相称的软件质量。而WebAssembly的目标是是通过暴露[底层能力][]接口给开发者，而不是规定哪些工具可以被构建来支持构建真正优秀的工具。底层能力可以：
* 移植现有或相似的工具到WebAssembly；
* 构建特别适合WebAssembly的新工具。

  [底层能力]: https://extensiblewebmanifesto.org

WebAssembly的开发要求能达到自托管（self-hosting）状态，成为一个愉开发者积极追求的愉快平台而不仅仅只是好看的黑客技术， 因为那些他们期望的开发工具*能够刚好满足期望*。 开发者具有很高的期望而满足这些工具上的期望意味着，WebAssembly具有为非开发者构建丰富应用的能力。

我们打算支持包括如下的工具：
* 编辑器：
  - 像vim和emacs这样的编辑器，要求达到*刚刚够用*。
* 编译器和语言级虚拟机：
  - 对于语言（C/C++，Rust，Go，C#）的编译目标是WebAssembly的编译器，要求能够在WebAssembly环境中运行，编译产生的WebAssembly模块能够在此环境立即运行。
  - 诸如bash，Python，Ruby这些语言的虚拟机，要求能够工作。
  - 具有just-in-time编译器的虚拟机（如JavaScript虚拟机，luajut，pypy），要求能提供以WebAssembly作为后端的新just-in-time运行环境。
* 调试器：
  - 通过对source map的支持，实现基本的浏览器集成。
  - 对于如C++等编程语言的完全集成，需要在调试信息格式，中断程序的权限，内省状态，修改状态等上进行更多的标准化工作。
  - 调试信息最好是能够按需提供，而不是内建于WebAssembly模块中。
* 对内存不安全语言的sanitizer支持：包括asan，tsan，msan，ubsan。对sanitizer的高效支持需要如下的改进：
  - 支持陷入（trap）；
  - shadow stack技术 (通常通过`mmap`的`MAP_FIXED`来实现)。
* 对开发者的代码具有可选的安全增强措施：WebAssembly开发者会希望他们的代码能被沙箱化，而这比WebAssembly的具体平台实现所要求提供的保护用户措施走的更远。
* 分析器：
  - 基于采样；
  - 基于测量。
* 进程dump：包括变量，调用栈，堆，全局变量，线程列表。
* JavaScript+WebAssembly大小优化工具：通过调用JavaScript来于Web平台的其他部分进行通信的庞大的WebAssembly+JavaScript混合应用程序，需要工具来进行dead code stripping*（译者注：删除目标文件中不需要加载的符号和代码）*和跨API边界的全局优化。

在许多的案例中，工具将会完全由无需任何特定WebAssembly工具支持的WebAssmbly实现；虽然这对调试器来讲是不可能的，但是对sanitizer来讲完全可行。
