# Polyfill到JavaScript

即使在浏览器提供WebAssembly的原生支持以前，开发者同样可以在Web上，使用将WebAssembly转换到JavaScript的[polyfill][]库来交付应用。除了将应用打包成前瞻性样式*（译者注：即二进制格式）*，以便于当浏览器原生地支持WebAssembly时，应用将立刻用上之外，polyfill同时提供了以下额外的好处：

* 采用[二进制编码](BinaryEncoding.md)传输，所以下载文件较小；
* 对启动性能影响较小（将二进制格式解码成JavaScript速度很快）；
* 与现有[Emscripten][]，[asm.js][]和[PNaCl][]等的将C/C++应用编译到浏览器的方法相比，具有相同的高吞吐率。

进一步的，polyfill将允许我们在早期的二进制编码上进行试验，同时在格式定稿之前获取开发者的反馈，然后作为[MVP版本](MVP.md)的一部分提供原生地支持。

在[polyfill仓库][]中，制定中的原型polyfill将拆开试验性的二进制格式到JavaScript，同时转换[asm.js][]到WebAssembly（对于现有的Web应用将会有用），我们同时也保留存在多个不同的polyfill以满足不同开发者的需求的可能性。

正如在浏览器嵌入的[实现细节](Web.md#implementation-details)中详细描述的一样，**高效**的polyfill会重用大量的Web平台现有的功能。

  [polyfill]: https://remysharp.com/2010/10/08/what-is-a-polyfill
  [Emscripten]: http://emscripten.org
  [asm.js]: http://asmjs.org
  [PNaCl]: http://gonacl.com
  [polyfill仓库]: https://github.com/WebAssembly/polyfill-prototype-1

## Polyfill的偏离

**高效**的polyfill可能会有意识地偏离具体的WebAssembly语义：这是为了在具体实践中可用，polyfill不必同WebAssembly规范保持100%的相同。在一些个别案例（通常是C/C++中未定义的行为）中，JavaScript和asm.js都没有理想的语义以高效地维护正确性。

如果有必要的话，polyfill可以在牺牲性能的条件下，提供选项用于保证语义的完全正确性，尽管对于可移植的C/C++代码来讲，这样的做法估计是不必的。

下面是一些我们已识别出的潜在且可取的语义偏离：

* **[非对齐的heap访问](AstSemantics.md#alignment)**：由于对非对齐的load/store操作需要保证返回正确结果但ams.js强制使用对齐（例如，`HEAP32[i>>2]`屏蔽了最末两位），所以asm.js实现的polyfill需要将*所有的*load/store操作转换到字节访问上（无论是否已经对齐）以保证正确。但为了达到有竞争力的性能，polyfill的原型将以全尺寸（full-size）的存取（看起来索引好像是对齐的）作为默认（但可能出错）方式。所以一般情况下，提供正确的对齐信息对于可移植WebAssembly的性能很重要；同时对齐信息也保证了polyfill正确和高效。
* **[heap访问越界](AstSemantics.md#out-of-bounds)**：无论WebAssembly行为如何，asm.js实现的polyfill将会遵循以下的标准asm.js行为：
  - 越界store操作将会被忽略（当作no-op操作）；
  - 对于整型的越界load操作，返回0；对于浮点型的越界load操作，返回NaN。
* **[32位整型操作](AstSemantics.md#32-bit-integer-operators)**：无论WebAssembly行为如何，asm.js实现的polyfill将会遵循以下的标准行为：
  - 除零返回0；
  - `INT32_MIN / -1`返回`INT32_MIN`；
  - 移位计数(shift count)将会被隐式屏蔽掉。
* **[数据类型的变换](AstSemantics.md#datatype-conversions-truncations-reinterpretations-promotions-and-demotions)**：无论WebAssembly行为如何，asm.js实现的polyfill将会遵循以下的标准行为：
  - 在将浮点数变换为整型失败时，返回0；
  - 可任意指定NaN的值。
* **[NaN的bit传播](AstSemantics.md#floating-point-operators)**：无论WebAssembly行为如何，asm.js实现的polyfill将会遵循以下的标准行为：
  - 可任意指定NaN的值。

## Polyfill的演变

整个MVP版本功能集将实现完全高效可polyfill化；在WebAssembly进化到MVP版本之后，工作组将会遵循如下原则：

* 标准化可以被polyfill的功能点；
* 其他的Web标准体协同进化， 以保证将来出现的WebAssembly功能点可被polyfill。

但是在MVP版本之后，并没有硬性的要求WebAssembly功能一定能被polyfill。如果出现某个非常可取但无法高效polyfill的功能，那么WebAssembly还是会采纳该功能。但这并不是一个可掉以轻心的决定，WebAssembly工作组将会试图使得开发者避免使用这些功能或者通过[功能检测](FeatureTest.md)机制提供fallback。
