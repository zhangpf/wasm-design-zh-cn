*（译者注：有些地方真的不知如何翻译，呼唤各路大神的帮助）*  
关于推动该功能的场景请查看[设计理念](Rationale.md#feature-testing---motivating-scenarios)文档

# 特性测试

在[PostMVP](PostMVP.md)版本中, 应用可以通过[`has_feature`或相似的API](PostMVP#Feature-Testing)来查询个特性是否被平台支持。该功能的初衷基于现实情况中，各个特性在不同的引擎里，会在不同的时间以不同的顺序被实现完成。

下面是描述特性测试功能的示例。

由于一些WebAssembly特性增加了操作，而所有的WebAssembly模块的代码都采用事先（ahead-of-time）的验证方式。常见的JavaScript的特性检测是如下模式：
```
if (foo)
    foo();
else
    alternativeToFoo();
```
而在WebAssembly中该代码将无法正常工作（如果 `foo` 函数不支持，那么`foo()` 将验证失败）。

在这种情况下，应用可以选择如下的两种策略之一作为替代方法：

1. 将模块编译成多个不同的版本，对应于多个不同平台所支持的特性，并使用`has_feature`方法测试决定将要加载的版本。

2. 在["特定"层解码](BinaryEncoding.md)阶段， which will happen
   in user code in the MVP *anyway*，使用`has_feature`来决定哪些特性已被支持， 并将不支持的功能映射到polyfill或陷入（trap）方法上实现。

上面两种做法都可使用工具链和编译flag来自动实现，并且`has_feature`是一个常量表达式，因此它可以被WebAssembly引擎常量合并。

为了说明这个问题，考虑下面四个例子：

* [`i32.min_s`](FutureFeatures.md#additional-integer-operators) —— 策略2可以被用来替换
  `(i32.min_s lhs rhs)`到等价的表达式
  ，即存储 `lhs` 和`rhs` 到局部变量中，然后使用`i32.lt_s` 和 `select`操作实现相同的功能。 
* [线程](PostMVP.md#threads) - 如果应用大量地使用`#ifdef`来进行线程开启/关闭的构建（build），那么策略1是适用的方法。但是，如果应用能够将线程的使用抽象成一些原语，策略2可以用来为正确的原语实现打补丁。
* [`mprotect`](FutureFeatures.md#finer-grained-control-over-memory) —— 如果引擎
  无法使用操作系统提供的signal handling机制来高效实现`mprotect`，那么`mprotect`可能永久地变成了一个可选特性。但如果`mprotect`的使用不是为了保证正确性（而仅仅是为了捕捉bug），即可用`nop`来替换。相反，为了保证正确性而使用`mprotect`但存在不依赖`mprotect`的替代方法，那么`mprotect`可以依靠应用测试`(has_feature "mprotect")` 方法来避免调用`abort()`，用`abort()`进行替换。`has_feature`查询操作可以通过现有的`__builtin_cpu_supports`函数暴露给C++代码。
* [SIMD](PostMVP.md#fixed-width-simd)（单指令多数据流）—— 当SIMD操作有足够优秀的polyfill实现时，例如通过`f32x4.mul`/`add`实现`f32x4.fma`指令, 那么策略2适用（与上面`i32.min_s`的例子相似）。相反的，当SIMD功能没有足够好的polyfill（例如，`f64x2`同时引入了操作符*和*类型）时，需要提供替代的算法并在加载时对其进行选择。

下面是一个假定（非真正实现）的对SIMD的`f64x2`功能进行polyfill的例子，C++编译器可以提供新的功能属性，来标识一个函数是另一个函数的优化，但特性依赖的版本(与
[`ifunc`属性](https://gcc.gnu.org/onlinedocs/gcc-4.7.2/gcc/Function-Attributes.html#index-g_t_0040code_007bifunc_007d-attribute-2529)相似，但不具有callback回调）：
```
#include <xmmintrin.h>
void foo(...) {
  __m128 x, y;           // -> f32x4 locals
  ...
  x = _mm_add_ps(x, y);  // -> f32x4.add
  ...
}
void foo_f64x2(...) __attribute__((optimizes("foo","f64x2"))) {
  __m256 x, y;           // -> f64x2 locals
  ...
  x = _m_add_pd(x, y);   // -> f64x2.add
  ...
}
...
foo(...);                 // calls either foo or foo_f64x2
```
在上面的例子中，工具链可以同时在“规范层”的二进制格式中，产生`foo`和`foo_f64x2`作为函数定义。如果`(has_feature "f64x2")`测试通过，那么polyfill会在加载时将`foo`替换成`foo_f64x2`。许多其他的策略也允许进行细粒度和粗粒度的替换，并由于这些替换都是发生在用户空间，所以策略可以随着时间不断演化。

请在[更好的特性测试支持](FutureFeatures.md#better-feature-testing-support)文档中查看进一步的功能。
