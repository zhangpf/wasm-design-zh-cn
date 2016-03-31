# WebAssembly设计文档，简体中文

> 本仓库包含了描述有关WebAssembly设计及其高层概述的文档。

> 本仓库的文档和讨论是[WebAssembly社区组](https://www.w3.org/community/webassembly/)内容的一部分。

## 概述

WebAssembly或者wasm，是一种新型可移植，具有占用存储小，加载速度快等特点的面向web应用的编译格式。

当前，WebAssembly是一项由[W3C社区组](https://www.w3.org/community/webassembly/)主导的开放标准，设计目标包括能被所有主流浏览器支持的格式等。*该仓库的内容处于不断变化：所有的内容仍然处于讨论之中*

- **WebAssembly是执行高效的**: wasm的[抽象语法树](AstSemantics.md)被设计为编码成占用存储小，加载时间短的[二进制格式](BinaryEncoding.md)。在设计目标上，通过利用现有平台普遍具有的[通用硬件能力](Portability.md#assumptions-for-efficient-execution)，WebAssembly可以达到接近原生二进制代码的执行速度。

- **WebAssembly是安全的**: WebAssembly描述了一个内存安全，沙箱式的[执行环境](AstSemantics.md#linear-memory)，该执行环境甚至可以利用现有的JavaScript虚拟机加以实现。当其内嵌于[Web](Web.md)环境时，WebAssembly将强制与浏览器同源并执行相同的权限安全策略以保证安全。

- **WebAssembly是开放和可调试的**: WebAssembly设计了美化的[文本格式](TextFormat.md)用于调试、测试、试验、优化、学习、传授以及直接使用该格式进行编程。该文本格式同时也在[查看wasm模块的代码](FAQ.md#will-webassembly-support-view-source-on-the-web)的功能中被使用。

- **WebAssembly是开放web平台体系中的一部分**: WebAssembly在设计上继续保持了无需更新、功能测试和向后兼容等[web应用的本质属性](Web.md)。WebAssembly模块可以与Javascript上下文互调用，并且与JavaScript使用相同的Web API来访问浏览器所有支持的功能。除此之外，WebAssembly同时支持[非web](NonWeb.md)的运行环境。

## 更多信息

| 资源							| 位置     |
|------------------|--------------------------|
| 高层设计目标 					| [HighLevelGoals.md](HighLevelGoals.md) |
| Frequently Asked Questions	| [FAQ.md](FAQ.md) |
| 语言规范(进行中......) 				| [spec/README.md](https://github.com/WebAssembly/spec) |
| 端到端的原型(从C++到浏览器虚拟机)	| [wasm-e2e/README.md](https://github.com/WebAssembly/wasm-e2e) |

## 设计流程和贡献

具有实际意义的WebAssembly规范是在[spec仓库](https://github.com/WebAssembly/spec/)中开展的。但目前为止，高层设计讨论将在继续在本设计仓库中，通过issue和pull request的方式进行，从而使得规范的制定工作更加聚焦。

我们计划wasm分步提供如下的功能：

 1. 在[最小可行性产品](MVP.md)；
 2. 接着[最小可行性产品之后](PostMVP.md)；
 3. 在[远期版本](FutureFeatures.md)中。

加入我们:

 * [W3C社区组](https://www.w3.org/community/webassembly/)
 * IRC频道: `irc://irc.w3.org:6667/#webassembly`
 * 通过[如何贡献](Contributing.md)页面!

如果您想贡献一份力，请查看[合乎道德和专业产品的代码](CodeOfConduct.md)页面。
