# RW_Spinlock 实现和测试指南

## 问题背景

在之前的性能测试中发现，即使启用了`USE_RW_SPINLOCK=1`，但实际的`BM_Find`测试仍然没有使用到rw_spinlock的并发读能力。原因是：

- `Lookup`函数调用的是`m_.find_fn()`（独占锁）
- 而不是`m_.find_fn_shared()`（共享锁）
- 这导致即使编译了rw_spinlock，实际运行时还是使用独占锁逻辑

## 需要修改的地方

### 1. 编译配置修改

#### 方法一：修改源码默认值（已完成）
```cpp
// 文件：cuckoohash_map.hpp 第1116行
#ifndef USE_RW_SPINLOCK
#define USE_RW_SPINLOCK 1  // 👈 改为1，启用rw_spinlock
#endif
```

#### 方法二：编译时传递宏定义（推荐用于对比测试）
```bash
# 编译时定义
bazel build //monolith/native_training/runtime/hash_table:hash_table_benchmark \
  --copt="-DUSE_RW_SPINLOCK=1"

# 或者在BUILD文件中添加
cc_binary(
    name = "hash_table_benchmark",
    srcs = ["hash_table_benchmark.cc"],
    copts = ["-DUSE_RW_SPINLOCK=1"],  # 👈 添加这行
    deps = [
        # ... 其他依赖
    ],
)
```

### 2. 核心调用链修改（关键！）

#### 修改Lookup函数（第165行）
```cpp
// 原代码 - 使用独占锁
int64_t Lookup(int64_t id, absl::Span<float> embedding) const override {
  auto find_fn = [&](EntryType& entry) {
    accessor_->Fill(entry_helper_.Get(entry), embedding);
  };
  if (m_.find_fn(id, find_fn)) {  // 👈 独占锁版本
    return 1;
  }
  std::memset(embedding.data(), 0, sizeof(float) * embedding.size());
  return 0;
}
```

```cpp
// 修改后 - 使用共享锁
int64_t Lookup(int64_t id, absl::Span<float> embedding) const override {
  auto find_fn = [&](EntryType& entry) {
    accessor_->Fill(entry_helper_.Get(entry), embedding);
  };
  if (m_.find_fn_shared(id, find_fn)) {  // 👈 共享锁版本
    return 1;
  }
  std::memset(embedding.data(), 0, sizeof(float) * embedding.size());
  return 0;
}
```

#### 修改LookupEntry函数（第179行）
```cpp
// 原代码 - 使用独占锁
void LookupEntry(int64_t id, absl::Span<EntryDump> entry) const override {
  auto find_fn = [&](EntryType& raw_entry) {
    entry[0] = std::move(accessor_->Save(entry_helper_.Get(raw_entry),
                                         raw_entry.GetTimestamp()));
  };
  if (m_.find_fn(id, find_fn)) {  // 👈 独占锁版本
    return;
  }
}
```

```cpp
// 修改后 - 使用共享锁
void LookupEntry(int64_t id, absl::Span<EntryDump> entry) const override {
  auto find_fn = [&](EntryType& raw_entry) {
    entry[0] = std::move(accessor_->Save(entry_helper_.Get(raw_entry),
                                         raw_entry.GetTimestamp()));
  };
  if (m_.find_fn_shared(id, find_fn)) {  // 👈 共享锁版本
    return;
  }
}
```

## 调用链分析

### 修改前的调用链（独占锁）
```
BM_Find benchmark
→ table->Lookup() 
→ CuckooEmbeddingHashTable::Lookup()
→ m_.find_fn(id, find_fn)  // 独占锁版本
→ cuckoohash_map::find_fn()
→ snapshot_and_lock_two<normal_mode>()  // 使用独占锁
→ spinlock::lock() 或 rw_spinlock::lock()  // 都是独占锁
```

### 修改后的调用链（共享锁）
```
BM_Find benchmark
→ table->Lookup() 
→ CuckooEmbeddingHashTable::Lookup()
→ m_.find_fn_shared(id, find_fn)  // 👈 共享锁版本
→ cuckoohash_map::find_fn_shared()
→ find_fn_shared_impl(..., std::true_type)  // enable_shared_lock=true
→ snapshot_and_lock_two_shared(hv)  // 使用共享锁
→ rw_spinlock::lock_shared()  // 👈 真正的并发读取！
```

## 实现细节

### rw_spinlock的共享锁实现
```cpp
class rw_spinlock {
private:
    // 使用32位原子变量存储状态
    // bit 31: 独占锁标志 (0x8000'0000)
    // bit 30: 写者等待标志 (0x4000'0000)  
    // bit 29-0: 读者计数 (0x3FFF'FFFF)
    std::atomic<std::uint32_t> state_;

public:
    // 获取共享锁（读锁）
    void lock_shared() noexcept {
        for (;;) {
            std::uint32_t st = state_.load(std::memory_order_relaxed);
            
            if (st < reader_lock_count_mask) {
                std::uint32_t newst = st + 1;  // 读者计数+1
                if (state_.compare_exchange_weak(st, newst, 
                    std::memory_order_acquire, std::memory_order_relaxed)) {
                    return;  // 成功获取共享锁
                }
            }
            // 继续自旋等待
        }
    }
    
    // 释放共享锁（读锁）
    void unlock_shared() noexcept {
        state_.fetch_sub(1, std::memory_order_release);  // 读者计数-1
    }
};
```

### find_fn_shared的条件编译
```cpp
template <typename K, typename F>
bool find_fn_shared(const K &key, F fn) const {
    return find_fn_shared_impl(key, fn, 
        std::integral_constant<bool, enable_shared_lock>{});
}

// 启用共享锁版本
template <typename K, typename F>
bool find_fn_shared_impl(const K &key, F fn, std::true_type) const {
    const hash_value hv = hashed_key(key);
    const auto b = snapshot_and_lock_two_shared(hv);  // 👈 使用共享锁
    const table_position pos = cuckoo_find(key, hv.partial, b.i1, b.i2);
    if (pos.status == ok) {
        fn(buckets_[pos.index].mapped(pos.slot));
        return true;
    }
    return false;
}

// 未启用共享锁，降级为独占锁
template <typename K, typename F>
bool find_fn_shared_impl(const K &key, F fn, std::false_type) const {
    return find_fn(key, fn);  // 降级为独占锁
}
```

## 已完成的修改

✅ **调用链修改**：
- `cuckoo_embedding_hash_table.cc` 第165行：`find_fn` → `find_fn_shared`
- `cuckoo_embedding_hash_table.cc` 第179行：`find_fn` → `find_fn_shared`

✅ **编译配置**：
- `cuckoohash_map.hpp` 第1116行：`USE_RW_SPINLOCK 0` → `USE_RW_SPINLOCK 1`

## 预期性能提升

### 理论分析
在读密集型工作负载（如`BM_Find`）中，rw_spinlock应该显示出显著优势：

```cpp
// 65536个细粒度锁的情况下
冲突概率 = 线程数 / 65536

// 即使发生冲突，多个读者可以并发
Thread A: rw_spinlock.lock_shared()  // 读者1
Thread B: rw_spinlock.lock_shared()  // 读者2 - 可以并发！
Thread C: rw_spinlock.lock_shared()  // 读者3 - 可以并发！
Thread D: rw_spinlock.lock_shared()  // 读者4 - 可以并发！
```

### 预期benchmark结果
```bash
# 修改前（使用find_fn，即使启用rw_spinlock也是独占锁）
BM_Find/10000000/1/32     6.02秒   (baseline)
BM_Find/10000000/4/32     ~5.8秒   (几乎无提升)
BM_Find/10000000/8/32     ~5.5秒   (提升有限)
BM_Find/10000000/32/32    ~4.8秒   (提升有限)

# 修改后（使用find_fn_shared，真正的并发读取）
BM_Find/10000000/1/32     6.02秒   (baseline)
BM_Find/10000000/4/32     ~1.5秒   (4倍提升)  
BM_Find/10000000/8/32     ~0.75秒  (8倍提升)
BM_Find/10000000/32/32    ~0.2秒   (30倍提升)
```

## 测试验证步骤

### 1. 编译和运行
```bash
# 编译当前版本（rw_spinlock + find_fn_shared）
bazel build //monolith/native_training/runtime/hash_table:hash_table_benchmark

# 运行benchmark
bazel run //monolith/native_training/runtime/hash_table:hash_table_benchmark
```

### 2. 对比测试
如果要对比spinlock vs rw_spinlock性能：

#### 步骤A：测试rw_spinlock（当前状态）
```bash
# 当前配置：USE_RW_SPINLOCK=1 + find_fn_shared
bazel run //monolith/native_training/runtime/hash_table:hash_table_benchmark > rw_spinlock_results.txt
```

#### 步骤B：切换到spinlock
```cpp
// 1. 修改 cuckoohash_map.hpp
#define USE_RW_SPINLOCK 0  // 改回0

// 2. 修改 cuckoo_embedding_hash_table.cc
if (m_.find_fn(id, find_fn)) {  // 改回find_fn
```

#### 步骤C：测试spinlock
```bash
bazel run //monolith/native_training/runtime/hash_table:hash_table_benchmark > spinlock_results.txt
```

#### 步骤D：对比结果
```bash
diff -u spinlock_results.txt rw_spinlock_results.txt
```

## 关键洞察

### 为什么之前rw_spinlock性能差？
1. **没有真正使用共享锁**：调用的是`find_fn`而不是`find_fn_shared`
2. **原子操作开销**：rw_spinlock比spinlock复杂，在独占模式下反而更慢
3. **细粒度锁架构**：65536个锁本身就降低了竞争，spinlock够用

### 修改后的优势
1. **真正并发读取**：多个线程可以同时持有读锁
2. **充分利用多核**：读操作不再串行化
3. **适合读密集型**：`BM_Find`正是读密集型工作负载

## 预期结果

修改完成后，在多线程的`BM_Find`测试中，rw_spinlock应该表现出显著的性能优势，特别是在高线程数场景下应该接近线性扩展。

这证明了读写锁在**真正的并发读取场景**下的价值，同时也解释了为什么之前的测试没有显示出预期的性能提升。
