## 块缓存

Block Cache 是 RocksDB 用于 **读操作** 的内存缓存，存储了从 SST 文件中读取的未压缩数据块，减少了对磁盘的访问

为了减轻锁争用问题，Block Cache 被划分为多个独立的分片，每个分片独立管理自己的缓存容量。默认情况下，每个缓存最多分片为 64 个分片，每个分片的最小容量为 512KB。

### LRUCache

- 哈希表：根据键（Key）和哈希值直接定位 Entries 在缓存中的位置，同时还记录 Entries 的位置（例如指向 LRU 链表节点的指针）。哈希表**仅负责查找和存储，不记录 Entries 使用的时间或顺序。**
- LRUCache：记录 Entries 的使用顺序，**动态更新 Entries 顺序**（每当 Entries 被访问时，从链表中移除该 Entries ，等 Realse 之后重新插入到 LRU 中）。通过节点链接，维护 Entries 之间的关系，实际的数据存储和快速定位由哈希表负责

启用 **cache_index_and_filter_blocks_with_high_priority （高优先级）**后，Block Cache 的 LRU 列表会被分为两部分：

- **高优先级池（High-pri Pool）**：存储索引块、过滤器块和压缩字典块。

- **低优先级池（Low-pri Pool）**：存储普通数据块。

高优先级池超出容量时，其尾部块会溢出到低优先级池，再与数据块竞争。

启用 **pin_l0_filter_and_index_blocks_in_cache** 后，将 **Level-0 层（L0 层）** 文件的索引块和过滤器块固定（pin）到 **Block Cache** 中，防止它们被逐出。L0 层中的文件通常是最新生成的文件，存储了最新的已刷盘数据，这些文件的数据块被读取的概率较高

启用 **pin_top_level_index_and_filter** 后，将分区索引和过滤器的 **顶层结构（存储了各个分区的元信息，用于快速定位分区内的数据块或过滤器） ** 固定到 Block Cache 中

```c++
// 分片缓存的单个分片
class LRUCacheShard final : public CacheShardBase {
  
  // 表示缓存的总容量
  size_t capacity_;

  // 当前高优先级池已使用的内存大小
  size_t high_pri_pool_usage_;

  // 当前低优先级池已使用的内存大小
  size_t low_pri_pool_usage_;

  // 当缓存达到最大容量时会拒绝新 Entries 的插入
  bool strict_capacity_limit_;
  // 高优先级池所占总容量的比例（0 到 1 的小数值）
  double high_pri_pool_ratio_;
  // 高优先级池的绝对容量
  double high_pri_pool_capacity_;

  double low_pri_pool_ratio_;
  double low_pri_pool_capacity_;

  // LRU 链表的虚拟头节点
  LRUHandle lru_;
  // 指向低优先级池在 LRU 链表中的头节点
  LRUHandle* lru_low_pri_;
  // 指向底层优先级池（最低优先级池）在 LRU 链表中的头节点
  LRUHandle* lru_bottom_pri_;
  // lru_.prev  --->  高优先级条目  ---> 
  // lru_low_pri_ --->  低优先级条目  ---> 
  // lru_bottom_pri_ ---> 超低优先级条目  ---> lru_.next
  
  // 存储所有缓存条目的哈希表
  LRUHandleTable table_;
  // 当前缓存中所有条目占用的内存总大小
  size_t usage_;
  // 当前 LRU 链表中条目占用的内存大小
  size_t lru_usage_;
};

// e       待插入的缓存条目
// handle  如果不为 nullptr，将被更新为插入的条目
Status LRUCacheShard::InsertItem(LRUHandle* e, LRUHandle** handle) {
  Status s = Status::OK();
  // 用于收集被逐出或释放的条目
  autovector<LRUHandle*> last_reference_list;

  {
    DMutexLock l(mutex_);
    // 按照严格的 LRU 策略逐出条目
    EvictFromLRU(e->total_charge, &last_reference_list);
    // 判断插入条目后是否超过缓存容量
    if ((usage_ + e->total_charge) > capacity_ &&
        (strict_capacity_limit_ || handle == nullptr)) {
      e->SetInCache(false);
      if (handle == nullptr) {
        // 不需要返回 handle
        // 条目不插入缓存，但将其视为立即被逐出，等待后续释放
        last_reference_list.push_back(e);
      } else {
        // 立即释放，避免调用者在资源未被释放的情况下试图访问条目数据
        free(e);
        e = nullptr;
        *handle = nullptr;
        s = Status::MemoryLimit("Insert failed due to LRU cache being full.");
      }
    } else {
      // 插入缓存
      LRUHandle* old = table_.Insert(e);
      usage_ += e->total_charge;
      if (old != nullptr) {
        // 如果插入时发现哈希表中已存在相同键的旧条目
        s = Status::OkOverwritten();
        old->SetInCache(false);
        if (!old->HasRefs()) {
          // 如果旧条目没有外部引用（引用计数为0），将其从 LRU 链表中移除，并减少内存使用量
          LRU_Remove(old);
          usage_ -= old->total_charge;
          last_reference_list.push_back(old);
        }
      }
      // 更新 LRU 链表
      if (handle == nullptr) {
        LRU_Insert(e);
      } else {
        if (!e->HasRefs()) {
          // 增加引用计数
          e->Ref();
        }
        *handle = e;
      }
    }
  }
  // 释放条目
  NotifyEvicted(last_reference_list);
  return s;
}

LRUHandle* LRUCacheShard::Lookup(const Slice& key, uint32_t hash,
                                 const Cache::CacheItemHelper* /*helper*/,
                                 Cache::CreateContext* /*create_context*/,
                                 Cache::Priority /*priority*/,
                                 Statistics* /*stats*/) {
  DMutexLock l(mutex_);
  LRUHandle* e = table_.Lookup(key, hash);
  if (e != nullptr) {
    assert(e->InCache());
    if (!e->HasRefs()) {
      // LRU 链表的职责是逐出未被引用的条目，而引用计数的职责是保护被外部使用的条目
      // 当条目被引用时，应切换到引用计数管理，从链表移除
      LRU_Remove(e);
    }
    e->Ref();
    e->SetHit();
  }
  return e;
}

// 释放一个缓存条目
// e                  缓存条目
// erase_if_last_ref  在条目引用计数归零时直接移除条目
bool LRUCacheShard::Release(LRUHandle* e, bool /*useful*/,
                            bool erase_if_last_ref) {
  if (e == nullptr) {
    return false;
  }
  bool must_free;
  bool was_in_cache;
  {
    DMutexLock l(mutex_);
    // 判断条目引用计数是否归零和是否仍在缓存中
    must_free = e->Unref();
    was_in_cache = e->InCache();
    if (must_free && was_in_cache) {
      if (usage_ > capacity_ || erase_if_last_ref) {
        // 简化逻辑，避免频繁的迁入迁出
        table_.Remove(e->key(), e->hash);
        e->SetInCache(false);
      } else {
        // 如果缓存未超出容量且未强制删除，将条目重新插入 LRU 链表
        LRU_Insert(e);
        must_free = false;
      }
    }
    // ... 释放条目
}

void LRUCacheShard::LRU_Insert(LRUHandle* e) {
  if (high_pri_pool_ratio_ > 0 && (e->IsHighPri() || e->HasHit())) {
    // 当明确标记为高优先级或者条目被命中过，便插入到高优先级的池子里
    e->next = &lru_;
    e->prev = lru_.prev;
    e->prev->next = e;
    e->next->prev = e;
    e->SetInHighPriPool(true);
    e->SetInLowPriPool(false);
    high_pri_pool_usage_ += e->total_charge;
    // 检查并维持高优先级池的大小限制
    MaintainPoolSize();
  } else if (low_pri_pool_ratio_ > 0 &&
             (e->IsHighPri() || e->IsLowPri() || e->HasHit())) {
    ...
  } else {
    ...
  }
  lru_usage_ += e->total_charge;
}

// 删除 LRU 链表最久未使用的条目，并同步从哈希表中删除这些条目
void LRUCacheShard::EvictFromLRU(size_t charge,
                                 autovector<LRUHandle*>* deleted) {
  while ((usage_ + charge) > capacity_ && lru_.next != &lru_) {
    // 从 LRU 链表的头部获取最久未使用的条目
    LRUHandle* old = lru_.next;
    LRU_Remove(old);
    table_.Remove(old->key(), old->hash);
    old->SetInCache(false);
    assert(usage_ >= old->total_charge);
    usage_ -= old->total_charge;
    // 将条目加入待释放列表
    deleted->push_back(old);
  }
}
```

### ClockCache

一种替代 LRUCache 的缓存实现，专为高并发环境下的性能优化而设计，在读操作占主导的场景下，HyperClockCache 的并发性能优于传统 LRUCache。适用于访问集中于少量数据的场景，例如索引块和过滤器块的缓存

- 大部分读操作（Lookup 和 Release）是单次原子操作，无需锁或等待，插入和删除（Insert 和 Eviction）支持并行操作，减少锁争用

- 基于增强型 CLOCK 算法，被频繁访问的 Entries 更难被删除
- 分片机制对更新性能（插入/删除）提升明显，但对读取性能影响较小
- 针对高频访问的数据路径（如索引块和过滤器块）进行了优化

##### 逐出算法（CLOCK 算法改进）

- 评分机制：
  - 每个 Entries 有一个初始评分（score），由其优先级决定。
  - 每次访问时评分增加，最高为 3；评分降为 0 且未被引用时， Entries 被逐出。

- 并行逐出：
  - 多线程可以同时扫描并更新 CLOCK 指针，支持高效逐出。
  - 每个线程处理多个槽位，减少锁争用。

##### 对引用计数的处理

- 仅未被引用的 Entries 会参与 CLOCK 算法操作。
- 插入或删除操作会根据引用状态调整 Entries 的可见性或逐出状态。

##### 元数据优化

- 引用计数 ： 表示 Entries 当前的外部使用次数
- 状态
  - **Empty**：槽位未被占用
  - **Construction**：正在插入或被释放
  - **Shareable**： 
    - **Visible**：Entries 可被 Lookup 命中
    - **Invisible**：Entries 被删除或不可见，但仍可被现有引用读取。

##### 状态转换机制

1. Empty -> Construction：插入新 Entries 时，原子操作确保槽位被独占
2. Construction -> Visible： 插入完成后，将 Entries 状态设置为Visible
3. Visible -> Invisible： 用户删除 Entries 时，将其设置为不可见，同时保持对现有引用的可访问性
4. Invisible -> Empty： Entries 引用计数归零后，释放槽位

##### 插入策略

- **避免覆盖**：
  - 插入时不会覆盖已有 Entries ，而是寻找空槽插入。
  - 旧 Entries 在不被使用时通过逐出机制移除。

- **Standalone Entries**：
  - 当缓存容量已满时，新 Entries 会作为 Standalone Entries 分配在堆上。
  - Standalone Entries 不会被后续查找命中，但可以正常使用，直到其引用计数归零。

```c++
// 固定大小的缓存表，支持高效的查找，适合静态容量管理。
// 采用开放寻址法，在发生哈希冲突时，使用一个次级哈希函数来计算步长，所有条目存储在哈希表本身
// 哈希表中的条目插入后位置是固定的，不能因后续插入或删除操作而移动
class FixedHyperClockTable : public BaseClockTable {

  // 默认负载为0.7，严格控制上限为0.84
  static constexpr double kLoadFactor = 0.7;
  static constexpr double kStrictLoadFactor = 0.84;
  
  // 控制哈希表大小： 1 << length_bits_.
  const int length_bits_;
  // 哈希表能存储的最大条目数量与 kLoadFactor 共同决定
  const size_t occupancy_limit_
  // 哈希表，存储元数据（如引用计数、状态标志）和缓存的数据指针。
  const std::unique_ptr<HandleImpl[]> array_;
};

FixedHyperClockTable::HandleImpl* FixedHyperClockTable::DoInsert(
    const ClockHandleBasicData& proto, uint64_t initial_countdown,
    bool keep_ref, InsertState&) {
  bool already_matches = false;
  HandleImpl* e = FindSlot(
      proto.hashed_key,
      [&](HandleImpl* h) {
        return TryInsert(proto, *h, initial_countdown, keep_ref,
                         &already_matches);
      },
      [&](HandleImpl* h) {
        if (already_matches) {
          // Stop searching & roll back displacements
          Rollback(proto.hashed_key, h);
          return true;
        } else {
          // Keep going
          return false;
        }
      },
      [&](HandleImpl* h, bool is_last) {
        if (is_last) {
          // Search is ending. Roll back displacements
          Rollback(proto.hashed_key, h);
        } else {
          h->displacements.FetchAddRelaxed(1);
        }
      });
  if (already_matches) {
    // Insertion skipped
    return nullptr;
  }
  if (e != nullptr) {
    // Successfully inserted
    return e;
  }
  
  assert(GetTableSize() < 256);
  return nullptr;
}

FixedHyperClockTable::HandleImpl* FixedHyperClockTable::Lookup(
    const UniqueId64x2& hashed_key) {
  HandleImpl* e = FindSlot(
      hashed_key,
      // 条目匹配逻辑
      [&](HandleImpl* h) {
        // 乐观查找
        constexpr bool kOptimisticLookup = true;
        uint64_t old_meta;
        if (!kOptimisticLookup) {
          old_meta = h->meta.Load();
          if ((old_meta >> ClockHandle::kStateShift) !=
              ClockHandle::kStateVisible) {
            return false;
          }
        }
        // 增加条目的引用计数，确保当前线程对条目有访问权限
        old_meta = h->meta.FetchAdd(ClockHandle::kAcquireIncrement);
        // 检查状态
        if ((old_meta >> ClockHandle::kStateShift) ==
            ClockHandle::kStateVisible) {
          // 状态为 Visible ，可能匹配
          if (h->hashed_key == hashed_key) {
            if (eviction_callback_) {
              // 如果存在缓存移除机制（CLOCK）
              // 命中需更新命中位
              h->meta.FetchOrRelaxed(uint64_t{1} << ClockHandle::kHitBitShift);
            }
            return true;
          } else {
            // 不匹配则释放
            Unref(*h);
          }
        } else if (UNLIKELY((old_meta >> ClockHandle::kStateShift) ==
                            ClockHandle::kStateInvisible)) {
          // 状态为 Invisible 则释放引用
          Unref(*h);
        } else {
          // 无需进一步操作
        }
        return false;
      },
      [&](HandleImpl* h) { return h->displacements.LoadRelaxed() == 0; },
      [&](HandleImpl* /*h*/, bool /*is_last*/) {});

  return e;
}

// 释放缓存条目
bool FixedHyperClockTable::Release(HandleImpl* h, bool useful,
                                   bool erase_if_last_ref) {
  // 相比于 LRU，当缓存超出容量且引用是最后一个时，不会删除 handle，
  // 空间仅由 EvictFromClock 和 Erase 释放，为了避免对 usage_ 进行额外的原子读取

  uint64_t old_meta;
  if (useful) {
    // 标记当前条目已经被成功使用，可以释放该引用
    // kReleaseIncrement 标识被释放的总次数
    old_meta = h->meta.FetchAdd(ClockHandle::kReleaseIncrement);
  } else {
    // 撤销之前错误或无效的引用操作，恢复条目的引用状态
    // kAcquireIncrement 表示被引用的总次数
    old_meta = h->meta.FetchSub(ClockHandle::kAcquireIncrement);
  }

  // 确保状态为 Shareable
  assert((old_meta >> ClockHandle::kStateShift) &
         ClockHandle::kStateShareableBit);
  // 确保引用计数不下溢出
  assert(((old_meta >> ClockHandle::kAcquireCounterShift) &
          ClockHandle::kCounterMask) !=
         ((old_meta >> ClockHandle::kReleaseCounterShift) &
          ClockHandle::kCounterMask));

  // 检查是否可以移除
  if (erase_if_last_ref || UNLIKELY(old_meta >> ClockHandle::kStateShift ==
                                    ClockHandle::kStateInvisible)) {
    // 更新引用计数
    if (useful) {
      old_meta += ClockHandle::kReleaseIncrement;
    } else {
      old_meta -= ClockHandle::kAcquireIncrement;
    }
    do {
      if (GetRefcount(old_meta) != 0) {
        // 如果引用计数不为0，说明条目仍在使用中，不能删除
        CorrectNearOverflow(old_meta, h->meta);
        return false;
      }
      if ((old_meta & (uint64_t{ClockHandle::kStateShareableBit}
                       << ClockHandle::kStateShift)) == 0) {
        // 如果条目已经不在 Shareable 状态，说明被其他线程获取，当前线程放弃操作
        return false;
      }
      // 通过 CAS 将状态从 Shareable 更新为 Construction
    } while (
        !h->meta.CasWeak(old_meta, uint64_t{ClockHandle::kStateConstruction}
                                       << ClockHandle::kStateShift));
  
    size_t total_charge = h->GetTotalCharge();
    if (UNLIKELY(h->IsStandalone())) {
      h->FreeData(allocator_);
      // 释放 standalone handle
      delete h;
      standalone_usage_.FetchSubRelaxed(total_charge);
      usage_.FetchSubRelaxed(total_charge);
    } else {
      // 普通条目的回收：回滚（撤销相关状态），释放资源，回收容量
      Rollback(h->hashed_key, h);
      FreeDataMarkEmpty(*h, allocator_);
      ReclaimEntryUsage(total_charge);
    }
    return true;
  } else {
    CorrectNearOverflow(old_meta, h->meta);
    return false;
  }
}


```

