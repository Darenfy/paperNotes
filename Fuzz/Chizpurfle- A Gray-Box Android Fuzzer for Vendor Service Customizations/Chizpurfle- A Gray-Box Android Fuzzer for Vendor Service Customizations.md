## Chizpurfle: A Gray-Box Android Fuzzer for Vendor Service Customizations


### 摘要
随着设备生产厂商定制化服务的出现，OS 设备可靠性和安全性会因为引入的软件漏洞而逐渐降低

由于厂商不提供源代码且服务不能跑在设备模拟器上，之前的工具并不能使用。本文提出了针对特定厂商安卓服务的灰盒模糊测试工具 Chizpurfle，该工具能够自动化发现、测试并描述专用服务

测试设备：Samsung Galaxy S6 Edge

### 简介
软件定制化可以使得用户拥有独一无二的经历和更愉悦的使用体验，但是会引入软件缺陷

由于供应商定制化不会整合到开源安卓中，无法从整个生态的反馈循环中受益，所以也会比核心 AOSP 代码库检查更粗心

供应商定制化是运行在特殊权限的代码，这也会使安全问题进一步恶化

白盒 Fuzz 技术不再适用于 Android 服务中

该工具利用动态二进制检测技术（例如软件断点和即时代码重写）的组合来获取有关块覆盖率的信息。

测试版本：Android 7
覆盖范围平均扩大 2.3 倍，最高 7.9 倍

### Background & Motivation
Vendor 软件定制化集中在三个方向：
- 设备驱动
- 预安装程序
- 系统服务

着重关注第三个方向，因为系统服务
- 通常运行于特权进程
- 直接暴露在用户应用程序面前
- 给低层接口提供包装器
- 代表了厂商定制化的大部分

通过对比，可以看出在设备中有大量可观的定制化服务与特定厂商的 Methods

由此可见，专属服务拥有庞大的攻击面和高级权限，需要特殊的工具验证其健壮性

Fuzz 测试需要根据测试覆盖率引导输入的产生

### Related Work
#### OS and systems software fuzzing
起初，fuzz 测试被广泛应用于测试系统软件，如网络服务器、shell 应用、库和 OS 内核

最现代化成熟的 fuzz 工具：AFL
- 基于在编译时修改目标程序的指令插桩 fuzzer
- 基于覆盖率测量，迭代提高 fuzz 输入的质量
- 通过使用 QEMU 模拟器无需编译插桩

白盒 fuzz 技术[符号执行]的代表：KLEE
- 基于 LLVM 的模拟器环境
- 仅在当目标运行在可以跟踪和派生符号状态的环境可应用

#### Android fuzzers
fuzzing Android system services
1. Buzzer
发送精心制作的 parcels
需要手动识别服务方法
2. BinderCracker
可自动理解 Binder 信息的格式，支持更复杂的通信模式

均是黑盒，不搜集任何信息，因此丢失提高 fuzz 效率的机会

### Tool Design
#### Architecture of Chizpurfle
![](截屏2020-11-19%20上午11.48.16.png)

**Orchestrator**
唯一一个运行在目标设备之外的部分
通过 ADB 加载和控制其他模块

- 由于 ADB 连接的不稳定，所以 Orchestrator 间断性连接并监测

- 将 Chizpurfle 运行在独立于 Zygote 的安卓运行时上，保证存活

**Method Extractor**
service-oriented architecture to manage several services
![](截屏2020-11-19%20下午2.07.54.png)

- 通过服务管理器获取所有已注册的服务列表，并迭代得到服务描述符列表
- 通过 Java 反射 API 得到接口定义，并得到服务中方法的特征

- 通过 hook ServiceManager 的调用，将每个服务映射到对应掌握服务的系统进程

**Instrumentation Module**
为了收集关于测试覆盖率的信息，IM 与运行目标服务的进程进行交互

requirements:
- 为了识别任何测试覆盖的新代码块，需要能够截断目标服务分支的执行

- 能够附加到系统进程

- 能够在实际设备上检测专属服务

解决方法：
- 硬件
利用用于调试的特殊的 CPU 特性
可惜不可利用?~！

- 软件
基于 Linux kernel 系统调用 ptrace
通常调试工具使用 ptrace 安装 软件断点

Chizpurfle 的检测跟踪机制
![](截屏2020-11-19%20下午2.47.19.png)

**Seed Manager**
负责给 Fuzz 输入生成器提供初始化输入

管理 seed 的优先级序列，基于输出分析器的分数，代表新执行块的数量

执行流代表着给进化算法提供了基础，能够驱动 fuzz 测试更深层次的目标服务

采用基于利用的常量计划表，每个种子只使用一次，种子消耗完，fuzz 结束

将多种算法应用于 fuzz 测试

Algorithm 1
![](截屏2020-11-19%20下午3.24.05.png)

**Fuzz 输入生成器**
依据变化的种子产生输入

产生的新输入的数量与种子的分数成比例，fuzz 操作符根据目标函数的参数类型确定

对应每种参数类型：
- 简单类型
- 字符串类型
- 数组、列表类型
- 对象类型

针对 Object 类型，提供了额外的特殊 fuzzer

**Test Executor**

使用目标服务相关的 IBinderObject 生成代理

Algorithm 2
![](截屏2020-11-19%20下午3.44.54.png)

**Output Analyzer**
分析日志识别由于 fuzz 触发的错误：
- A/F messages
- ANR messages
- FATAL messages

注意：
1. 关注于系统进程的日志而不是 Test Executor
2. 注册回调函数 linkToDeath，当服务进程死亡时获取通知

Algorithm 3
![](截屏2020-11-19%20下午3.56.29.png)

**Futher Optimizations**
需要解决的技术问题：  
系统服务执行在系统进程的上下文中，与数十个其他线程一同，同时监测所有线程耗费大量资源，降低 fuzz 测试的速度

- 不监测与服务无关的线程
基于启发式算法探测无关线程
  - 标记服务名称，重新获取标记令牌
  - 获取进程的线程名称
  - 对比识别线程名称和令牌名称

- 避免小部分 false positives
周期性监测电量水平，如果电量不足，则暂停测试

### Evaluation Campaign
**三星定制化中引入的漏洞**
number of Service Methods: 2272
number of tests: 34645
number of caused failures: 9
number of vul: 2

**与黑盒 Fuzz 比较**
performance overhead：11.97x slow-down per service
test coverage: 2.3x more code

