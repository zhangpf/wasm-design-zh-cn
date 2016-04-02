# 二进制编码

本文档描述了WebAssembly模块的[可移植](Portability.md)二进制格式。

二进制编码是模块信息的密集表示方式，并同时达到减小文件体积，快速解码以及减少内存使用量的目的。更多信息请查看[设计理念](Rationale.md#why-a-binary-encoding)部分。

编码的过程被分为如下三层：

* **第0层（Layer 0）**是抽象语法树（AST）以及相关的数据结构的一个简单的前序编码。编码的结果是密集且容易交互的，使得其适合于JIT，测量工具，调试等场景。
* **第1层（Layer 1）**利用了语法树和节点本身的特定知识，对第0层结果进行结构性压缩。结构压缩方法引入了更多高效的值编码、模块内部的重排序以及剔除结构相同树节点等方法。
* **第2层（Layer 2）**进一步使用了多种通用压缩算法，例如[gzip](http://www.gzip.org/)和[Brotli](https://datatracker.ietf.org/doc/draft-alakuijala-brotli/)等已经在现有浏览器和其它工具中实现的算法。

更重要的是，这种分层方法使得开发和标准制定可以增量地进行。例如，通过在应用层上解压缩到更低层的方式，可以试验第1层和第2层编码技术。压缩技术一旦稳定下来以后，那么将成为标准并转移到原生的实现上。

# 数据类型

### uint8
单字节无符号整型

### uint32
四字节[小端序](https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82%E5%BA%8F#.E5.B0.8F.E7.AB.AF.E5.BA.8F)（little endian）无符号整型

### varint32
[有符号LEB128格式](https://en.wikipedia.org/wiki/LEB128#Signed_LEB128)32位可变长整型.

### varuint32
[无符号LEB128格式](https://en.wikipedia.org/wiki/LEB128)的32位可变长整型， `varuint32`的值可能包含前导0。

### varint64
有符号LEB128格式64位可变长整型。

### 值类型
使用一个单字节无符号整型表示[值类型](AstSemantics.md#types)。这些类型具体编码如下：
* `1`表示`i32`类型
* `2`表示`i64`类型
* `3`表示`f32`类型
* `4`表示`f64`类型


# 定义

### 前序编码
依据语法树的编码方式，每个节点以可识别二进制序列开始，接着是所有子节点的循环编码。

* 示例
  * 给定一个简单的AST节点：`I32Add(left: AstNode, right: AstNode)`
    * 首先对`I32Add`的编码（uint8类型）
    * 然后依次循环编码左节点和右节点

  * 给定一个调用AST节点：`Call(callee_index: uint32_t, args: AstNode[])`
    * 首先对`Call`的编码（uint8类型）
    * 然后写入（可变长）整型`callee_index`的值（varuint32类型）
    * 然后依次循环编码每个参数变量节点， 其中参数列表长度通过在签名列表中查询`callee_index`得到

# 模块结构

本节记录的是当前模块结构的原型格式。该格式是以[原生v8上的wasm原型格式](https://docs.google.com/document/d/1-G11CnMA0My20KI9D7dBR6ZCPOBCRD0oCH6SHCPFGx0/edit?usp=sharing)为基础，并将在未来取代之。

## 高层结构

模块以如下形式的魔术数字（magic number）和版本号开头

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| magic number | `uint32` | `0x6d736100` (也就是'\0asm') |
| version | `uint32` | 版本号。当前是10，MVP将会重置版本为1 |

接着是一系列的段。通常情况下段是可重复的, 但是某些类型的段只能出现一次，或某些段所依赖的其它段必须出现在其之前，但又不必挨着该段出现，所以可以以任意先后顺序出现。每一段以一个立即字符串作为标识（ID），而对于某些段，如果它的标识符（ID）对于某个具体的WebAseembly实现是未知的，那么这些段将会被忽略。所有的段的以如下的内容作为编码开始：

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| size  | `varuint32` | 该段（除了本字段）的大小，以字节为单位 |
| id_len | `varuint32` | 该段的标识符长度 |
| id_str | `bytes` | 长度为id_len字节的标识符字符串 |

### 内存段

ID： `memory`

内存段声明了与该模块相关联的内存的大小和特征。一个模块最多只有一个内存段。

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| min_mem_pages | `varuint32` | 最小可能内存大小，以64KiB页为单位 |
| max_mem_pages | `varuint32` | 最小可能内存大小，以64KiB页为单位 |
| exported | `uint8` | 如果内存对对模块外可见，则为`1` |

### 签名段

ID： `signatures`

签名段声明了该模块会用到的所有函数的签名。一个模块最多只有一个签名段。

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| count | `varuint32` | 签名项的数量 |
| entries | `signature_entry*` | 签名项列表，具体描述如下 |

#### 签名项
| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| param_count | `varuint32` | 函数的参数数量 |
| return_type | `value_type?` | 函数的返回类型，如果该项是`0`则表示无返回值 |
| param_types | `value_type*` | 函数参数的类型 |

### 导入表段

ID： `import_table`

导入表段声明了该模块会用到的所有导入项。一个模块最多只有一个导入表段。

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| count | `varuint32` | 导入项的数量 |
| entries | `import_entry*` | 导入项列表，具体描述如下 |

#### 导入项
| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| sig_index | `varuint32` | 该导入项的在函数表的索引 |
| module_len | `varuint32` | 被导入的模块名称的长度 |
| module_str | `bytes` | 长度为`module_len`的模块名称字符串 |
| function_len | `varuint32` | 被导入的函数名称长度 |
| function_str | `bytes` | 长度为`function_len`的函数名称字符串 |

### 函数签名段

ID： `function_signatures`

函数签名段声明了该模块所有函数的签名，并且该段必须位于[签名段](#签名段)之后。一个模块最多只有一个函数签名段。

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| count | `varuint32` | 函数签名项的数量 |
| signatures | `varuint32*` | 该函数签名项出现在签名表的位置的下标组成的列表 |

### 函数体段

ID： `function_bodies`

函数体段为模块中的每个函数指定了其主体部分的内容，因此该段必须出现在[函数签名段](#函数签名段)之后。
函数签名项的数量和函数体项的数量必须相同，并且第`i`个签名项必须对应于第`i`个函数主体。

| Field | Type |  Description |
| ----- |  ----- |  ----- |  ----- |
| count | `varuint32` | 函数体项的数量 |
| bodies | `function_body*` | [函数体](#函数体)的列表 |

### 导出表段

ID： `export_table`

导出表段声明了该模块所有导出项。一个模块最多只有一个导出表段，且该段必须位于[函数签名段](#函数签名段)之后。

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| count | `varuint32` | 导出项的数量 |
| entries | `import_entry*` | 导出项列表，具体描述如下 |

#### 导出项
| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| func_index | `varuint32` | 该项在函数表出现的位置索引 |
| function_len | `varuint32` | 函数名长度 |
| function_str | `bytes` | 长度为`function_len`的函数名称字符串编码 |

### 开始函数段

ID： `start_function`

一个模块至多有一个函数开始段，并且该段必须出现在[函数签名段](#函数签名段)之后。

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| index | `varuint32` | 开始函数在函数表的索引 |

### 数据段段

ID： `data_segments`

数据段段声明了已初始化并将加载到线性内存的数据。一个模块最多可能有一个数据段段。

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| count | `varuint32` | 数据段项的数量 |
| entries | `data_segment*` | 数据段项列表，具体描述如下 |

`data_segment`内容如下：

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| offset | `varuint32` | 存储该数据的位置，以在线性存储中的位移量作为表示 |
| size | `varuint32` | `data`的大小，以字节为单位 |
| data | `bytes` | `size`个字节的具体数据 |

### 间接函数表段

ID： `function_table`

间接函数表段声明了间接函数表的大小和每个表项的值，该值是函数在[函数签名段](#函数签名段)所在位置的索引（因此该段也必须出现在函数签名段之后）。

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| count | `varuint32` | 间接函数表项的数量 |
| entries | `varuint32*` | 间接函数表项列表，以在在函数表位置索引作为每一项 |

### 名字段

ID： `names`

名字段在模块中可能出现0或1次，并且不会改变执行时语义。该段在验证时出现错误不会导致模块验证失败，而是判定该段在模块中不存在。名字段的期望使用场景是，当二进制的WebAssembly模块在浏览器或其它开发环境中被查看时，该段的内容将被作为函数名和局部名，出现在[文本格式](TextFormat.md)中。

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| count | `varuint32` | 名字项的数量 |
| entries | `function_names*` | 名字项列表 |

`function_name`列表中每一项对在函数项列表中同一索引的函数项指派名字。其数量可能会不等于实际函数的数量。

#### 函数名

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| fun_name_len | `varuint32` | 字符串长度，以字节为单位 |
| fun_name_str | `bytes` | 合法的utf8字符串编码 |
| local_count | `varuint32` | 局部名项数量 |
| local_names | `local_name*` | 局部名项列表 |

`local_name`列表每一项对在局部项列表中同一索引的局部项指派名字为。其数量可能会不等于实际的局部值数量

#### 局部名

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| local_name_len | `varuint32` | 字符串长度，以字节为单位 |
| local_name_str | `bytes` | 合法的utf8字符串编码 |

### 结束段

ID： `end`

该段表示模块的结束，任何在本段之后出现的多余数据都不会被解码器解释。（正是由于不会被解释，结束段之后部分）可以用来，例如，存储函数名或数据段主体内容。

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |

### 未知段

| 字段 | 类型 | 描述 |
| ----- |  ----- | ----- |
| body  | `bytes` | 该段的内容 |

如果某段的ID对于WebAssembly的具体实现是未知的，那么该段（是未知段）并会被忽略。

# 函数体

函数体包含了局部变量声明序列，然后接着是[抽象语法树](AstSemantics.md)的密集前序编码结果。抽象语法树的每个节点对应于一个操作，例如`i32.add`、`if`或`block`。每个操作节点被编码成操作码，接着是立即数（如果有），再接着子节点（如果有）的形式。

| 名字 | 操作码 | 描述 |
| ----- | ----- | ----- |
| body size | `varuint32` | 函数体的大小，以字节为单位 |
| local count | `varuint32` | 局部项的数量 |
| locals | `local_entry*` | 局部变量的列表 |
| ast    | `byte*` | 前序编码后的AST |

#### 局部项

每一个局部项声明了数个同一类型的局部变量，并允许多个局部项具有相同的变量类型。

| 字段 | 类型 | 描述 |
| ----- | ----- | ----- |
| count | `varuint32` | 变量的数量 |
| type | `value_type` | 变量的类型 |


## 控制流操作([具体描述](AstSemantics.md#control-flow-structures))

| 操作名 | 操作码 | 立即数 | 描述 |
| ---- | ---- | ---- | ---- |
| `nop` | `0x00` | | 空操作 |
| `block` | `0x01` | count = `varuint32` | 表达式序列，该序列最后一个表达式产生一个返回值 |
| `loop` | `0x02` | count = `varuint32` | 产生控制流回路的块 |
| `if` | `0x03` | | 高层单臂if（即只有if，没有else） |
| `if_else` | `0x04` | | 高层双臂if（即if-else） |
| `select` | `0x05` | | 依照条件选择两个值中的一个 |
| `br` | `0x06` | relative_depth = `varuint32` | 以嵌套该块的外部块为目标进行跳转 |
| `br_if` | `0x07` | relative_depth = `varuint32` | 以嵌套该块的外部块为目标进行条件跳转 |
| `br_table` | `0x08` | 解释如下 | 分支表控制流结构 |
| `return` | `0x14` | | 从函数中返回值或返回0（表示无返回值） |
| `unreachable` | `0x15` | | 立即陷入（trap） |

`br_table`操作有一个立即操作数，该操作数采用如下编码方式：

| 字段 | 类型 | 描述 |
| ---- | ---- | ---- |
| target_count | `varuint32` | target_table中目标项个数 |
| target_table | `uint32*` | 目标项列表，每一项表示可以跳转到外部块或环路的目标位置 |
| default_target | `uint32` | 默认跳转到外部块或环路的目标位置 |

`br_table`操作码实现了间接分支功能，它接受一个`i32`表达式值作为输入的位移量，在`target_table`范围内跳转到目标块或环路。如果输入值超出了范围，那么`br_table`将跳转到默认的目标位置。

## 基本操作（[详细描述](AstSemantics.md#constants)）
| 名称 | 操作码 | 立即数 | 描述 |
| ---- | ---- | ---- | ---- |
| `i32.const` | `0x0a` | value = `varint32` | 作为`i32`类型的常量 |
| `i64.const` | `0x0b` | value = `varint64` | 作为`i32`类型的常量 |
| `f64.const` | `0x0c` | value = `uint64` | 作为`u64`类型的常量 |
| `f32.const` | `0x0d` | value = `uint32` | 作为`u32`类型的常量 |
| `get_local` | `0x0e` | local_index = `varuint32` | 读局部变量或参数的值 |
| `set_local` | `0x0f` | local_index = `varuint32` | 为局部变量或参数赋值 |
| `call` | `0x12` | function_index = `varuint32` | 根据索引（index）调用函数 |
| `call_indirect` | `0x13` | signature_index = `varuint32` | 根据期望的签名值间接调用函数 |
| `call_import` | `0x1f` | import_index = `varuint32` | 根据索引（index）调用被导入的函数 |

## 内存相关的操作（[详细描述](AstSemantics.md#linear-memory-accesses)）

| 名称 | 操作码 | 立即数 | 描述 |
| ---- | ---- | ---- | ---- |
| `i32.load8_s` | `0x20` | `memory_immediate` | 从内存加载 |
| `i32.load8_u` | `0x21` | `memory_immediate` | 从内存加载  |
| `i32.load16_s` | `0x22` | `memory_immediate` | 从内存加载 |
| `i32.load16_u` | `0x23` | `memory_immediate` | 从内存加载 |
| `i64.load8_s` | `0x24` | `memory_immediate` | 从内存加载 |
| `i64.load8_u` | `0x25` | `memory_immediate` | 从内存加载 |
| `i64.load16_s` | `0x26` | `memory_immediate` | 从内存加载 |
| `i64.load16_u` | `0x27` | `memory_immediate` | 从内存加载 |
| `i64.load32_s` | `0x28` | `memory_immediate` | 从内存加载 |
| `i64.load32_u` | `0x29` | `memory_immediate` | 从内存加载 |
| `i32.load` | `0x2a` | `memory_immediate` | 从内存加载 |
| `i64.load` | `0x2b` | `memory_immediate` | 从内存加载 |
| `f32.load` | `0x2c` | `memory_immediate` | 从内存加载 |
| `f64.load` | `0x2d` | `memory_immediate` | 从内存加载 |
| `i32.store8` | `0x2e` | `memory_immediate` | 存入内存 |
| `i32.store16` | `0x2f` | `memory_immediate` | 存入内存 |
| `i64.store8` | `0x30` | `memory_immediate` | 存入内存 |
| `i64.store16` | `0x31` | `memory_immediate` | 存入内存 |
| `i64.store32` | `0x32` | `memory_immediate` | 存入内存 |
| `i32.store` | `0x33` | `memory_immediate` | 存入内存 |
| `i64.store` | `0x34` | `memory_immediate` | 存入内存 |
| `f32.store` | `0x35` | `memory_immediate` | 存入内存 |
| `f64.store` | `0x36` | `memory_immediate` | 存入内存 |
| `memory_size` | `0x3b` |  | 查询内存大小（*译者注：应该是分配的主存空间大小*） |
| `grow_memory` | `0x39` |  | 增加内存大小 |

`memory_immediate`类型的编码方式如下：

| 名称 | 类型 | 描述 |
| ---- | ---- | ---- |
| flags | `varuint32` | bit字段，当前是在最低有效位上包含对齐信息（alignment）， 以`log2(alignment)`作为编码 |
| offset | `varuint32` | 位移值 |

正如`log2(alignment)`所隐式表达的，对齐长度必须是2的次幂。并且作为额外的验证要求，对齐长度不得大于自然对齐长度。`log(memory-access-size)`最低有效位之后所有位必须置0。而这些位被保留以便未来使用（例如，留作共享内存定序用）。

## 简单操作 ([详细描述](AstSemantics.md#32-bit-integer-operators))

| 名称 | 操作码 | 立即数 | 描述 |
| ---- | ---- | ---- | ---- |
| `i32.add` | `0x40` | | |
| `i32.sub` | `0x41` | | |
| `i32.mul` | `0x42` | | |
| `i32.div_s` | `0x43` | | |
| `i32.div_u` | `0x44` | | |
| `i32.rem_s` | `0x45` | | |
| `i32.rem_u` | `0x46` | | |
| `i32.and` | `0x47` | | |
| `i32.or` | `0x48` | | |
| `i32.xor` | `0x49` | | |
| `i32.shl` | `0x4a` | | |
| `i32.shr_u` | `0x4b` | | |
| `i32.shr_s` | `0x4c` | | |
| `i32.rotr` | `0xb6` | | |
| `i32.rotl` | `0xb7` | | |
| `i32.eq` | `0x4d` | | |
| `i32.ne` | `0x4e` | | |
| `i32.lt_s` | `0x4f` | | |
| `i32.le_s` | `0x50` | | |
| `i32.lt_u` | `0x51` | | |
| `i32.le_u` | `0x52` | | |
| `i32.gt_s` | `0x53` | | |
| `i32.ge_s` | `0x54` | | |
| `i32.gt_u` | `0x55` | | |
| `i32.ge_u` | `0x56` | | |
| `i32.clz` | `0x57` | | |
| `i32.ctz` | `0x58` | | |
| `i32.popcnt` | `0x59` | | |
| `i32.eqz`  | `0x5a` | | |
| `i64.add` | `0x5b` | | |
| `i64.sub` | `0x5c` | | |
| `i64.mul` | `0x5d` | | |
| `i64.div_s` | `0x5e` | | |
| `i64.div_u` | `0x5f` | | |
| `i64.rem_s` | `0x60` | | |
| `i64.rem_u` | `0x61` | | |
| `i64.and` | `0x62` | | |
| `i64.or` | `0x63` | | |
| `i64.xor` | `0x64` | | |
| `i64.shl` | `0x65` | | |
| `i64.shr_u` | `0x66` | | |
| `i64.shr_s` | `0x67` | | |
| `i64.rotr` | `0xb8` | | |
| `i64.rotl` | `0xb9` | | |
| `i64.eq` | `0x68` | | |
| `i64.ne` | `0x69` | | |
| `i64.lt_s` | `0x6a` | | |
| `i64.le_s` | `0x6b` | | |
| `i64.lt_u` | `0x6c` | | |
| `i64.le_u` | `0x6d` | | |
| `i64.gt_s` | `0x6e` | | |
| `i64.ge_s` | `0x6f` | | |
| `i64.gt_u` | `0x70` | | |
| `i64.ge_u` | `0x71` | | |
| `i64.clz` | `0x72` | | |
| `i64.ctz` | `0x73` | | |
| `i64.popcnt` | `0x74` | | |
| `i64.eqz`  | `0xba` | | |
| `f32.add` | `0x75` | | |
| `f32.sub` | `0x76` | | |
| `f32.mul` | `0x77` | | |
| `f32.div` | `0x78` | | |
| `f32.min` | `0x79` | | |
| `f32.max` | `0x7a` | | |
| `f32.abs` | `0x7b` | | |
| `f32.neg` | `0x7c` | | |
| `f32.copysign` | `0x7d` | | |
| `f32.ceil` | `0x7e` | | |
| `f32.floor` | `0x7f` | | |
| `f32.trunc` | `0x80` | | |
| `f32.nearest` | `0x81` | | |
| `f32.sqrt` | `0x82` | | |
| `f32.eq` | `0x83` | | |
| `f32.ne` | `0x84` | | |
| `f32.lt` | `0x85` | | |
| `f32.le` | `0x86` | | |
| `f32.gt` | `0x87` | | |
| `f32.ge` | `0x88` | | |
| `f64.add` | `0x89` | | |
| `f64.sub` | `0x8a` | | |
| `f64.mul` | `0x8b` | | |
| `f64.div` | `0x8c` | | |
| `f64.min` | `0x8d` | | |
| `f64.max` | `0x8e` | | |
| `f64.abs` | `0x8f` | | |
| `f64.neg` | `0x90` | | |
| `f64.copysign` | `0x91` | | |
| `f64.ceil` | `0x92` | | |
| `f64.floor` | `0x93` | | |
| `f64.trunc` | `0x94` | | |
| `f64.nearest` | `0x95` | | |
| `f64.sqrt` | `0x96` | | |
| `f64.eq` | `0x97` | | |
| `f64.ne` | `0x98` | | |
| `f64.lt` | `0x99` | | |
| `f64.le` | `0x9a` | | |
| `f64.gt` | `0x9b` | | |
| `f64.ge` | `0x9c` | | |
| `i32.trunc_s/f32` | `0x9d` | | |
| `i32.trunc_s/f64` | `0x9e` | | |
| `i32.trunc_u/f32` | `0x9f` | | |
| `i32.trunc_u/f64` | `0xa0` | | |
| `i32.wrap/i64` | `0xa1` | | |
| `i64.trunc_s/f32` | `0xa2` | | |
| `i64.trunc_s/f64` | `0xa3` | | |
| `i64.trunc_u/f32` | `0xa4` | | |
| `i64.trunc_u/f64` | `0xa5` | | |
| `i64.extend_s/i32` | `0xa6` | | |
| `i64.extend_u/i32` | `0xa7` | | |
| `f32.convert_s/i32` | `0xa8` | | |
| `f32.convert_u/i32` | `0xa9` | | |
| `f32.convert_s/i64` | `0xaa` | | |
| `f32.convert_u/i64` | `0xab` | | |
| `f32.demote/f64` | `0xac` | | |
| `f32.reinterpret/i32` | `0xad` | | |
| `f64.convert_s/i32` | `0xae` | | |
| `f64.convert_u/i32` | `0xaf` | | |
| `f64.convert_s/i64` | `0xb0` | | |
| `f64.convert_u/i64` | `0xb1` | | |
| `f64.promote/f32` | `0xb2` | | |
| `f64.reinterpret/i64` | `0xb3` | | |
| `i32.reinterpret/f32` | `0xb4` | | |
| `i64.reinterpret/f64` | `0xb5` | | |
