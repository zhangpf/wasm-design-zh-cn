# 最小可行性产品

正如在[高层设计目标](HighLevelGoals.md)中提到的，WebAssembly第一个发布版本的目标是实现最小可行性产品（MVP）。这意味着某些我们*已知的*功能需求将在MVP之后出现，对于这部分功能，我们单列了一个[post-MVP](PostMVP.md)功能的计划文档。MVP将包含现有web浏览器已经具有并且在移动设备上运行良好的功能，这些大致和[asm.js](http://asmjs.org)所具有的功能相同。

MVP主要部件的设计被拆分为如下几个文档：

* 在WebAssembly中，可分发、加载和执行的代码单元被称为[模块](Modules.md)。
* 一个模块中WebAssembly代码的行为，按照其[抽象语法树](AstSemantics.md)进行规范。
* 被设计为能被WebAssembly原生地解码，WebAssembly的二进制格式，被规范化为模块的抽象语法树的[二进制序列化](BinaryEncoding.md)。 
* 能够使用工具（如汇编器，调试器，分析器）进行读和写操作的WebAssembly文本格式，被规范化为模块抽象语法树的[文本化映射](TextFormat.md)。
* 在设计上，WebAssembly将在[Web浏览器](Web.md)和[非浏览器执行环境](NonWeb.md)中同时实现。
* 为了简化迁移到WebAssembly的同时保持对现有平台的支持，WebAssembly的MVP版本还将实现一个高效的[polyfill库](Polyfill.md)。

