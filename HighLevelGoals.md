# WebAssembly高层设计目标

1. 定义一种[可移植](Portability.md), 占用体积小和加载速度快的[二进制格式](MVP.md)作为编译的目标格式，并利用现有平台（包括[移动设备](https://en.wikipedia.org/wiki/Mobile_device)和[IoT设备](https://en.wikipedia.org/wiki/Internet_of_Things)）普遍具有的通用硬件能力，使其达到原生代码的执行速度。
2. 逐步规范和实现：
    * 一个大致与[asm.js](http://asmjs.org)具有相同功能的[最小可行性产品(MVP)](MVP.md)，其主要的目标语言是[C/C++](CAndC++.md)；
    * 针对MVP，实现一个将WebAssembly转换成JavaScript的高效[polyfill](Polyfill.md)库，以便于WebAssembly的MVP版本能在现有的浏览器中运行；
    * [MVP后续版本](PostMVP.md)将在MVP基础上增加一些必不可少的功能；
    * 在此基础上逐步细化，以及根据反馈和经验附加[新功能](FutureFeatures.md)，包括支持除C/C++之外的其它语言。
3. WebAssembly能在*现有的*[Web平台](Web.md)执行，并与这些平台良好的集成：
    * 保持无需更新、[功能已测试](FeatureTest.md)和
      [向后兼容](BinaryEncoding.md)等Web发展的成果；
    * 与JavaScript在相同的语义领域下执行；
    * 允许异步地与JavaScript互调用；
    * 强制与浏览器同源并执行相同的权限安全策略；
    * 通过与JavaScript使用相同的Web API来访问浏览器功能；并定义了一种可以与二进制格式相互转换的易编辑文本格式以支持查看源代码功能。
4. 在设计上，WebAssembly同样支持[非浏览器嵌入](NonWeb.md)。
5. 构建一个优秀的平台:
    * 为WebAssembly构建一个新的LLVM后端，同时伴随一个clang的移植版本([为什么首先是LLVM？](FAQ.md#which-compilers-can-i-use-to-build-webassembly-programs))；
    * 鼓励发展其它针对于WebAseembly的编译器和工具; 并且
    * 实现其它实用[工具](Tooling.md).
