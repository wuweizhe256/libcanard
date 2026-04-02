# Libcanard 漏洞植入计划

本计划概述了如何将复杂的释放后使用 (Use-After-Free, UAF) 逻辑缺陷植入 `libcanard` 代码库中，特别是针对 RX 会话的内存管理逻辑。该修改将伪装为“性能迭代：Rx 会话对象标记与对象池”的常规更新。

## 提议的变更

---

### 核心数据平面的修改 (`canard.h` 和 `canard.c`)

我们将以此名义引入一个会话后备缓存，旨在“减少高并发下的内存碎片”。
- **`canard.h`**：在 `struct CanardInstance` 结构体尾部添加 `void* session_cache;`。
- **`canard.c`**：
  - 在 `CanardInternalRxSession` 结构体中添加 `void* cache_next;` 以实现链表池。
  - 修改 `canardRxUnsubscribe()` 以缓存会话对象，而不是调用 free 完全析构它。已分配的 `payload.data`（报文数据缓冲区）会被正常释放 (`deallocate`) 以节省大型内存池空间，但在此过程中该指针字段将**故意不被**置为 `NULL`。
  - 修改 `rxAcceptFrame()`，在收到新传输需分配新会话之前，优先检查 `ins->session_cache`。当从缓存中弹出旧会话复用时，初始化逻辑将“意外地”遗漏对 `rxs->payload.data` 和 `rxs->payload.size` 的清零操作，隐式地继承（信任）了原本对象的残留状态。

### 元数据记录 (`metadata/vulnerability_rca.json`)

我们将 Root Cause (漏洞根因) 的真实追踪路径和 Concept Proof (POC) 触发原理解析写入单独的 `metadata` 目录，以确保 `libcanard` 代码主库的纯净，不留任何测试注释痕迹。
