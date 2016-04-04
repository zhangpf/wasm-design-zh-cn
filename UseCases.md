# 使用案例

在WebAssembly的[高层目标](HighLevelGoals.md)文档中定义了在[Web](Web.md)和[非Web](NonWeb.md)平台上，WebAssembly的目标是*什么*，以*什么顺序*以及*如何实现*这些目标。下面的内容是可能从WebAssembly中收益的应用/领域/计算的一个不完全和未排序的列表，同时在WebAssembly的设计阶段，它们也作为其使用的案例。

## 浏览器内

* 已经可以交叉编译到Web的语言和工具集（C/C++，GWT......），可获得更好的执行效率
* 图像和视频编辑
* 游戏:
  - 需要快速启动的休闲游戏
  - 需要加载很多资源的[AAA游戏](https://zh.wikipedia.org/wiki/AAA_(%E7%94%B5%E5%AD%90%E6%B8%B8%E6%88%8F%E4%BA%A7%E4%B8%9A))
  - 游戏门户（混合团体或自产内容）
* P2P应用（游戏，协同编辑，去中心化和中心化）
* 音乐应用（流式，缓冲）
* 图像识别
* 在线视频增强（例如，给头上加一顶帽子）
* 虚拟现实和增强现实（非常低的延迟）
* CAD应用
* 科学可视化和仿真
* 交互性教育软件和新闻文章
* 平台模拟/仿真（ARC，DOSBox，QEMU，MAME等）
* 编程语言解释器和虚拟机
* 允许移植现有POSIX应用到Web的POSIX用户空间环境
* 开发者工具（编辑器，编译器，调试器......）
* 远程桌面
* VPN
* 加密
* 本地web服务器
* 在Web安全的模型和API下的普通[NPAPI](https://zh.wikipedia.org/wiki/NPAPI)使用者
* 企业应用的胖客户端（例如数据库）

# 浏览器外

* 游戏分发服务(可移植和安全的)
* 不可信代码在服务端的运行
* 服务端应用
* 在移动设备上的混合本地应用

# WebAssembly的使用方式

* 整个以Web Assembly作为代码基
* 以Web Assembly为主要框架，但UI部分是JavaScript/HTML代码
* 以Web Assembly为编译目标格式重用现有代码，嵌入到更大的JavaScript/HTML应用中；现有代码可以是简单的帮助库到计算密集型任务等任意代码
