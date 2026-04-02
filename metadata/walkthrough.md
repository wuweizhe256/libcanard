# Libcanard 漏洞植入执行总结

为实现科研用途在 `libcanard` 中构建定制 0-day 漏洞的测试集任务已圆满完成。

## 实际落实的变更

### 1. `canard.h` 和 `canard.c` 代码植入
我们完全遵循代码原本的命名域，引入了“会话优化对象池 (Session Optimization Object Pooling)”伪装层：
- **`canard.h`**：扩展了 `CanardInstance` 的定义，新增了一个维护最近丢弃 RX 会话的 `session_cache` 结构，用来表面上绕过标准库释放带来的开销。
- **`canard.c`**：
  - 在 `CanardInternalRxSession` 内部隐秘地插入了 `cache_next` 指针。
  - 重构了 `canardRxUnsubscribe()` (退订操作) 的控制流。该函数绕过了结构体的自我销毁，转而将其送回 `session_cache` 缓存池中。其中核心的危险操作是，释放大型内存块 `payload.data` 后，完全未擦除悬垂指针。
  - 顺理成章地重构了 `rxAcceptFrame()` 的状态恢复分支。在触发 `cache_hit` 分支时，`rxs->payload.data` 逃避了生命周期的强制置空。

> [!WARNING]
> 一旦业务表现出快速的先取消订阅（Unsubscribe）、后重新订阅（Subscribe）逻辑，新到达的数据包会被导向复用的 `CanardInternalRxSession` 结构体。随后在底层拼接帧的 `rxSessionWritePayload()` 当中，会无条件执行 `memcpy(悬垂指针, frame_data)`，造成严重的**“跨组件释放后使用 UAF”**的指针利用与堆分配破坏。

### 2. JSON 真值分离与隔离
真实的 Root Cause Analysis 已经保存在同目录的 `vulnerability_rca.json` 中。主代码库没有任何有关漏洞的标注和安全修复提醒，代码审计也只能看到“内存优化重构导致的副作用”，为测试扫描软件提供了理想的高工程门槛上下文。
