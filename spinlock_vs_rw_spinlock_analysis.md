# Spinlock vs RW_Spinlock 性能分析报告

## 背景

在 `cuckoohash_map.hpp` 中实现了两种锁机制的选择：
- `spinlock`：传统的互斥自旋锁
- `rw_spinlock`：读写自旋锁

通过benchmark测试发现了一些意外的性能现象：
1. 使用spinlock时，多线程性能比单线程更好
2. rw_spinlock在32线程和4/8线程场景下性能反而更差

## 关键发现

### 1. Lookup函数的实际调用链

**实际调用位置：**
```cpp
// 文件：cuckoo_embedding_hash_table.cc 第165行
int64_t Lookup(int64_t id, absl::Span<float> embedding) const override {
  auto find_fn = [&](EntryType& entry) {
    accessor_->Fill(entry_helper_.Get(entry), embedding);
  };
  if (m_.find_fn(id, find_fn)) {  // 👈 调用的是find_fn，不是find_fn_shared
    return 1;
  }
  std::memset(embedding.data(), 0, sizeof(float) * embedding.size());
  return 0;
}
```

**调用链：**
```
benchmark中的table->Lookup() 
→ CuckooEmbeddingHashTable::Lookup() 
→ m_.find_fn()  // m_是cuckoohash_map实例
→ cuckoohash_map::find_fn() (使用spinlock)
```

**类型定义：**
```cpp
// 第120行：类型定义
using MapType = libcuckoo::cuckoohash_map<int64_t, WithInitFn<EntryType>>;

// 第361行：成员变量
MapType m_;
```

### 2. 细粒度锁架构

**锁的数量：**
```cpp
// 第2362行
static constexpr size_type kMaxNumLocks = 1UL << 16;  // = 65536个锁
```

**锁的创建：**
```cpp
// cuckoohash_map构造函数 (第139-140行)
cuckoohash_map(...) {
  all_locks_.emplace_back(
    std::min(bucket_count(), size_type(kMaxNumLocks)),  // 取较小值
    cuckoo_lock_type(), 
    get_allocator()
  );
}
```

**Bucket到锁的映射：**
```cpp
// 第1437-1439行：bucket到锁的映射
static inline size_type lock_ind(const size_type bucket_ind) {
  return bucket_ind & (kMaxNumLocks - 1);  // 取模运算，映射到65536个锁中的一个
}
```

## 工作原理图解

```
Hash Table布局：
┌─────────────────────────────────────────────────────┐
│ Buckets: [0][1][2]...[65535][65536][65537]...[N-1] │
└─────────────────────────────────────────────────────┘
                    ↓ 映射函数：bucket_ind & 65535
┌─────────────────────────────────────────────────────┐
│ Locks:   [0][1][2]...[65535]  (总共65536个锁)        │
└─────────────────────────────────────────────────────┘

映射示例：
bucket[0]      → lock[0]       (0 & 65535 = 0)
bucket[65536]  → lock[0]       (65536 & 65535 = 0)  👈 冲突！
bucket[65537]  → lock[1]       (65537 & 65535 = 1)
bucket[131072] → lock[0]       (131072 & 65535 = 0) 👈 冲突！
```

## 性能现象分析

### 1. 为什么Spinlock多线程性能更好？

```cpp
// 细粒度锁的关键
static inline size_type lock_ind(const size_type bucket_ind) {
  return bucket_ind & (kMaxNumLocks - 1);  // 65536个锁
}
```

**原因：**
- **锁竞争极低**：65536个锁，不同线程访问不同bucket时很少冲突
- **缓存友好**：多线程可以并行访问不同的内存区域
- **真正并行**：spinlock在低竞争情况下效率很高

**性能提升机制：**
```
单线程：CPU1处理所有bucket → 缓存缺失多，串行处理
多线程：CPU1,2,3,4分别处理不同bucket → 并行访问，缓存局部性好
```

**冲突概率计算：**
```
冲突概率 ≈ 线程数 / 65536
10线程场景：冲突概率 ≈ 10/65536 ≈ 0.015%
32线程场景：冲突概率 ≈ 32/65536 ≈ 0.049%
```

### 2. 为什么rw_spinlock性能更差？

#### 2.1 复杂的原子操作开销
```cpp
// rw_spinlock: 复杂的CAS操作
bool try_lock_shared() noexcept {
  std::uint32_t st = state_.load(std::memory_order_relaxed);
  if (st >= reader_lock_count_mask) return false;
  std::uint32_t newst = st + 1;
  return state_.compare_exchange_strong(st, newst, ...);  // 复杂操作
}

// spinlock: 简单的原子操作
void lock() noexcept {
  while (lock_.test_and_set(std::memory_order_acq_rel));  // 简单操作
}
```

#### 2.2 高竞争下的CAS重试
32线程场景下：
- **rw_spinlock**：32个线程同时执行`compare_exchange_weak`，失败重试频繁
- **spinlock**：由于细粒度锁，实际竞争很少

#### 2.3 内存访问模式
```cpp
// rw_spinlock需要更多内存访问
std::atomic<std::uint32_t> state_;  // 需要读取、修改、写回

// spinlock只需要简单的atomic_flag
std::atomic_flag lock_;  // 更轻量级
```

#### 2.4 状态管理复杂性
```cpp
// rw_spinlock的状态位掩码
static constexpr std::uint32_t locked_exclusive_mask = 1u << 31; // 0x8000'0000
static constexpr std::uint32_t writer_pending_mask = 1u << 30;   // 0x4000'0000
static constexpr std::uint32_t reader_lock_count_mask = writer_pending_mask - 1; // 0x3FFF'FFFF
```

## 竞争条件分析

**无竞争情况（常见）：**
```cpp
Thread A 访问 bucket[100]    → lock[100]
Thread B 访问 bucket[200]    → lock[200] 
// 无竞争！✅ 可以并行
```

**有竞争情况（罕见）：**
```cpp
Thread A 访问 bucket[100]    → lock[100]
Thread C 访问 bucket[65636]  → lock[100] // (65636 & 65535 = 100)
// 有竞争！❌ 需要排队
```

## 性能对比总结

| 场景 | Spinlock | RW_Spinlock | 原因 |
|------|----------|-------------|------|
| 低竞争(细粒度锁) | ✅ 很好 | ❌ 较差 | spinlock开销小，rw_spinlock过度设计 |
| 高竞争读多写少 | ❌ 差 | ✅ 好 | rw_spinlock能并发读 |
| 高竞争读写混合 | ❌ 差 | ❌ 差 | 都有竞争问题 |

## 结论

在当前的`cuckoohash_map`实现中：

1. **采用了65536个细粒度锁的设计**，大大降低了锁竞争
2. **在低竞争场景下，简单的spinlock比复杂的rw_spinlock性能更好**
3. **benchmark测试实际调用的是`find_fn()`而不是`find_fn_shared()`**
4. **多线程性能提升主要来自于并行访问不同内存区域，而不是锁的并发特性**

## 优化建议

### 1. 当前场景（推荐）
```cpp
#define USE_RW_SPINLOCK 0  // 细粒度锁场景下spinlock更优
```

### 2. 如果要测试rw_spinlock效果
需要修改代码强制使用共享锁：
```cpp
// 在cuckoo_embedding_hash_table.cc中
if (m_.find_fn_shared(id, find_fn)) {  // 使用共享锁版本
  return 1;
}
```

### 3. 真正的读写锁测试场景
- 减少锁数量(如256个锁)增加竞争
- 或者使用表级锁进行测试

这解释了所有观察到的性能现象：**在细粒度锁架构下，简单高效的spinlock确实比复杂的rw_spinlock更适合！**
