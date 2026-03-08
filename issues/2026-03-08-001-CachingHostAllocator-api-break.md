# [2026-03-08-001] CachingHostAllocator API 兼容性问题

- **发现日期**：2026-03-08
- **编号**：2026-03-08-001
- **严重级别**：🔴 编译失败
- **受影响文件**：
  - `torch_npu/csrc/core/npu/CachingHostAllocator.cpp`
- **触发版本**：PyTorch nightly 2026-03-08
- **对应 patch**：`patches/0001-fix-CachingHostAllocator-api-compat.patch`

---

## 问题描述

构建在编译 `CachingHostAllocator.cpp` 阶段失败，涉及 NPU 扩展内存分配器模块。错误类型为类型不兼容和成员不存在。

---

## 根本原因分析

### 1. BlockPool 名称冲突

PyTorch 上游在 `CachingHostAllocatorImpl` 中新增了 `BlockPool` typedef：

```cpp
template <typename S_, typename E_, typename B_>
struct HostBlockPool {
  std::mutex blocks_mutex_;
  ska::flat_hash_set<B_*> blocks_;  // 注意：blocks_ 而非 blocks
  // ... 没有 unmapped 成员
};

// 基类中的 typedef
using BlockPool = HostBlockPool<S, E, B>;
```

Ascend 侧定义了同名的 `BlockPool` 结构体，导致类型冲突：

```cpp
struct BlockPool {  // 与基类 typedef 冲突
    std::set<ExpandableBlock *, Comparison> blocks;
    std::set<ExpandableBlock *, Comparison> unmapped;  // 上游已移除
};
```

### 2. HostBlockPool 成员变更

上游 `HostBlockPool::blocks` 重命名为 `blocks_`，且移除了 `unmapped` 成员。

### 3. process_events() 签名变更

基类 `process_events()` 签名变更为 `void process_events(BlockPool& pool)`，而 Ascend 侧定义为 `void process_events() override`。

关键错误日志：
```
error: 'void at_npu::native::NPUExpandableHostAllocatorImpl::process_events()' marked 'override', but does not override
error: 'struct at::HostBlockPool<...>' has no member named 'blocks'; did you mean 'blocks_'?
error: 'struct at::HostBlockPool<...>' has no member named 'unmapped'
```

---

## 修复方案

见 `patches/0001-fix-CachingHostAllocator-api-compat.patch`，核心改动：

1. **重命名 BlockPool 为 ExpandableBlockPool**：避免与基类 typedef 冲突
2. **移除 process_events() 的 override 标记**：因为签名已变更
3. **保持 ExpandableBlockPool 的独立实现**：继续使用 `blocks` 和 `unmapped` 成员，因为这是 Ascend 特有的扩展内存功能

> 注意事项：此 patch 仅解决名称冲突问题，ExpandableBlockPool 作为 Ascend 特有的数据结构保持独立实现。
