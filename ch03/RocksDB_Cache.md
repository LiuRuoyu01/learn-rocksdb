## Block Cache

Block Cache 是 RocksDB 用于 **读操作** 的内存缓存，存储了从 SST 文件中读取的未压缩数据块，减少了对磁盘的访问

为了减轻锁争用问题，Block Cache 被划分为多个独立的分片，每个分片独立管理自己的缓存容量。默认情况下，每个缓存最多分片为 64 个分片，每个分片的最小容量为 512KB。

### LRUCache

- 哈希表：根据键（Key）和哈希值直接定位 Entries 在缓存中的位置，同时还记录 Entries 的位置（例如指向 LRU 链表节点的指针）。哈希表**仅负责查找和存储，不记录 Entries 使用的时间或顺序**。通过单链表来解决 hash 冲突，每次针对 hash 表的链表插入都会采用头插法，保证越新的 key 越靠近 bucket 头部
- LRUCache：双链表，记录 Entries 的使用顺序，**动态更新 Entries 顺序（每当 Entries 被访问时，从链表中移除该 Entries ，等 Realse 之后重新插入到 LRU 中）** 。通过节点链接，维护 Entries 之间的关系，实际的数据存储和快速定位由哈希表负责

LRU 存在三种状态

1. 外部引用（refs >= 1），在哈希表中（in_cache == true）：entry 不在 LRU 表中
2. 未被外部引用（refs == 0），在哈希表中（in_cache == true）： entry 在 LRU 表，可以被 free
3. 外部引用（refs >= 1），不在哈希表中（in_cache == false）：entry 即不在 LRU 也不在 Hash 表，如果在此状态下 refs 变为 0 则必须被 free 

状态变化：

- 状态 1 to 状态 2：Release
- 状态 1 to 状态 3：Erase 或 Insert 一个相同的 key
- 状态 2 to 状态 1：LookUp

启用 **cache_index_and_filter_blocks_with_high_priority （高优先级）** 后，Block Cache 的 LRU 列表会被分为两部分：

- **高优先级池（High-pri Pool）**：存储索引块、过滤器块和压缩字典块。

- **低优先级池（Low-pri Pool）**：存储普通数据块。

高优先级池超出容量时，其尾部块会溢出到低优先级池，再与数据块竞争。

启用 **pin_l0_filter_and_index_blocks_in_cache** 后，将 **Level-0 层（L0 层）** 文件的索引块和过滤器块固定（pin）到 **Block Cache** 中，防止它们被逐出。L0 层中的文件通常是最新生成的文件，存储了最新的已刷盘数据，这些文件的数据块被读取的概率较高

启用 **pin_top_level_index_and_filter** 后，将分区索引和过滤器的 **顶层结构（存储了各个分区的元信息，用于快速定位分区内的数据块或过滤器）** 固定到 Block Cache 中

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
    // 按照严格的 LRU 策略逐出旧数据，保证 e 有足够空间存储
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
    // ... 
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
  - 插入时不会覆盖已有 Entries 。
    - 覆盖旧 Entries 会存在读写冲突（在修改 Entries 时仍有线程在读取该条目），锁竞争等
  - 旧 Entries 在不被使用时通过逐出机制移除。
- **Standalone Entries**：
  - 当缓存容量已满时，新 Entries 会作为 Standalone Entries 分配在堆上。
  - Standalone Entries 不会被后续查找命中，但可以正常使用，直到其引用计数归零。

```c++
Status BaseClockTable::Insert(const ClockHandleBasicData& proto,
                              typename Table::HandleImpl** handle,
                              Cache::Priority priority, size_t capacity,
                              uint32_t eec_and_scl) {
  
  if (eec_and_scl & kStrictCapacityLimitBit) {
    // kStrictCapacityLimitBit 情况下，如果内存满且移除条目失败会返回错误状态
    Status s = ChargeUsageMaybeEvictStrict<Table>(
        total_charge, capacity, need_evict_for_occupancy, eec_and_scl, state);
    if (!s.ok()) {
      occupancy_.FetchSubRelaxed(1);
      return s;
    }
  } else {
    bool success = ChargeUsageMaybeEvictNonStrict<Table>(
        total_charge, capacity, need_evict_for_occupancy, eec_and_scl, state);
    if (!success) {
      occupancy_.FetchSubRelaxed(1);
      if (handle == nullptr) {
        // 非 kStrictCapacityLimitBit 情况下，内存满且移除条目失败
        // 如果 handle 为空，默认条目已被插入并立即驱逐
        proto.FreeData(allocator_);
        return Status::OK();
      } else {
        // 需要通过独立插入 standalone_insert
        usage_.FetchAddRelaxed(total_charge);
        use_standalone_insert = true;
      }
    }
  }

  if (!use_standalone_insert) {
    // 使用标准插入，如果找到相同条目则放弃插入
    HandleImpl* e =
        derived.DoInsert(proto, initial_countdown, handle != nullptr, state);
    if (e) {
      if (handle) {
        *handle = e;
      }
      return Status::OK();
    }
    // 插入失败会进行回退操作
    occupancy_.FetchSubRelaxed(1);
    if (handle == nullptr) {
      // 如果 handle 为空，默认该条目已经被插入但被立即驱逐
      usage_.FetchSubRelaxed(total_charge);
      assert(usage_.LoadRelaxed() < SIZE_MAX / 2);
      proto.FreeData(allocator_);
      return Status::OK();
    }
    // 执行独立插入
    use_standalone_insert = true;
  }
  *handle = StandaloneInsert<HandleImpl>(proto);
  return Status::OkOverwritten();
}

HandleImpl* BaseClockTable::StandaloneInsert(
    const ClockHandleBasicData& proto) {
  // 分配在堆上，处理具体数据和元数据
  HandleImpl* h = new HandleImpl();
  ClockHandleBasicData* h_alias = h;
  *h_alias = proto;
  // Standalone Entries 指的是那些没有成功插入到主表中的条目
  // 通常这些条目会被单独管理，不会参与主表的替换
  h->SetStandalone();
  // 设置状态为 Invisible，不会被后续查找命中
  uint64_t meta = uint64_t{ClockHandle::kStateInvisible}
                  << ClockHandle::kStateShift;
  meta |= uint64_t{1} << ClockHandle::kAcquireCounterShift;
  h->meta.Store(meta);
  return h;
}

```

```c++
/*
low bits                                                     high bits
----------------------------------------------------------------------
| acquire counter      | release counter     | hit bit | state marker |
----------------------------------------------------------------------
acquire counter:  每次条目被 Lookup() 或其他类似操作获取时，该计数器会增加
release counter:  这个计数器表示条目被释放的次数。每次引用结束时，release counter 会递增,过比较 acquire counter 和 release counter,当两者相等时，条目处于未被引用的状态，表明它可以被考虑驱逐
hit bit:          通常用来标记某个缓存条目是否近期有被访问
state marker:     标记条目的状态
*/

// 固定大小的缓存表，支持高效的查找，适合静态容量管理。
// 采用开放寻址法，在发生哈希冲突时，使用一个次级哈希函数来计算步长，所有条目存储在哈希表本身
// 哈希表中的条目插入后位置是固定的，不能因后续插入或删除操作而移动
// 当 entry 被删除时会留下空洞，通过记录位移计数器 displacements 解决此问题
// displacements 记录了有多少个元素原本应该位于该槽位或更低的位置
// FindSlot 通过 displacements 做查找的终止条件
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
        // 尝试讲条目插入到当前的槽中，如果存在会更新 already_matches
        return TryInsert(proto, *h, initial_countdown, keep_ref,
                         &already_matches);
      },
      [&](HandleImpl* h) {
        if (already_matches) {
          // 如果条目已经存在当前槽中则需要回滚
          Rollback(proto.hashed_key, h);
          return true;
        } else {
          return false;
        }
      },
      [&](HandleImpl* h, bool is_last) {
        if (is_last) {
          // 撤销之前对 displacements 的修改
          Rollback(proto.hashed_key, h);
        } else {
          // 记录状态变化
          h->displacements.FetchAddRelaxed(1);
        }
      });
 
  //...
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
              // 需更新命中位
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
// erase_if_last_ref:   如果当前条目是最后一个引用且需要删除时，可以进行删除操作
// 相比于 LRU，当缓存超出容量且引用是最后一个时，不会删除 handle，
// 空间仅由 EvictFromClock 和 Erase 释放
bool FixedHyperClockTable::Release(HandleImpl* h, bool useful,
                                   bool erase_if_last_ref) {
  uint64_t old_meta;
  if (useful) {
    // 标记当前条目已经被成功使用，可以释放该引用
    old_meta = h->meta.FetchAdd(ClockHandle::kReleaseIncrement);
  } else {
    // 撤销之前错误或无效的引用操作，恢复条目的引用状态
    old_meta = h->meta.FetchSub(ClockHandle::kAcquireIncrement);
  }

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
      // 释放 standalone handle
      h->FreeData(allocator_);
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
    // 防止引用计数溢出
    CorrectNearOverflow(old_meta, h->meta);
    return false;
  }
}

void FixedHyperClockTable::Erase(const UniqueId64x2& hashed_key) {
  (void)FindSlot(
      hashed_key,
      [&](HandleImpl* h) {
        // 获取 Acquire 计数器
        uint64_t old_meta = h->meta.FetchAdd(ClockHandle::kAcquireIncrement);
        if ((old_meta >> ClockHandle::kStateShift) ==
            ClockHandle::kStateVisible) {
          if (h->hashed_key == hashed_key) {
            // 更改状态为 Invisible，以便后续查找不可见
            old_meta =
                h->meta.FetchAnd(~(uint64_t{ClockHandle::kStateVisibleBit}
                                   << ClockHandle::kStateShift));
            old_meta &= ~(uint64_t{ClockHandle::kStateVisibleBit}
                          << ClockHandle::kStateShift);
            for (;;) {
              uint64_t refcount = GetRefcount(old_meta);
              if (refcount > 1) {
                // 该条目在删除过程中还有其他引用，因此不删除它，撤销对该条目的引用
                Unref(*h);
                break;
              } else if (h->meta.CasWeak(
                             old_meta, uint64_t{ClockHandle::kStateConstruction}
                                           << ClockHandle::kStateShift)) {
                // 通过 CAS 将状态改为 Construction，表示该条目正在被删除中
                size_t total_charge = h->GetTotalCharge();
                FreeDataMarkEmpty(*h, allocator_);
                ReclaimEntryUsage(total_charge);
                Rollback(hashed_key, h);
                break;
              }
            }
          } else {
            // 不匹配，则撤销对条目的引用
            Unref(*h);
          }
        } else if (UNLIKELY((old_meta >> ClockHandle::kStateShift) ==
                            ClockHandle::kStateInvisible)) {
          // 如果条目的状态是 Invisible，表示该条目已经被删除，直接撤销引用
          Unref(*h);
        } else {
          // 其他状态下，条目不再被查找或引用，不会改变条目的可用性或引用计数
        }
        return false;
      },
      // displacements 用于判断该条目是否在操作过程中被干扰，是否可以安全地进行删除或其他操作
      [&](HandleImpl* h) { return h->displacements.LoadRelaxed() == 0; },
      [&](HandleImpl* /*h*/, bool /*is_last*/) {});
}

template <typename MatchFn, typename AbortFn, typename UpdateFn>
inline FixedHyperClockTable::HandleImpl* FixedHyperClockTable::FindSlot(
    const UniqueId64x2& hashed_key, const MatchFn& match_fn,
    const AbortFn& abort_fn, const UpdateFn& update_fn) {
  size_t base = static_cast<size_t>(hashed_key[1]);
  // 确保增量值与表大小互质（即不共享因子），避免陷入死循环
  size_t increment = static_cast<size_t>(hashed_key[0]) | 1U;
  size_t first = ModTableSize(base);
  size_t current = first;
  bool is_last;
  do {
    HandleImpl* h = &array_[current];
    if (match_fn(h)) {
      return h;
    }
    if (abort_fn(h)) {
      return nullptr;
    }
    // 多次探测直到探测完所有的槽
    current = ModTableSize(current + increment);
    is_last = current == first;
    update_fn(h, is_last);
  } while (!is_last);

  return nullptr;
}

```



# Table Cache

Table Cache 通过 Row Cache （热点数据直接存在内存中）和 Table Reader （将已打开的 SST 文件的数据缓存在内存中）平衡内存和磁盘的访问，解决 I/O 瓶颈

**Row Cache --> Table Cache --> Block Cache**

```c++
// options                            : 本次读取的策略
// internal_comparator                : internal_key 的比较逻辑
// file_meta                          : SST 文件的元数据
// k                                  : 需要读取的 key
// skip_filters                       : 是否跳过加载布隆过滤器
// level                              : 标识 SST 文件所存在的 LSM 层级
// max_file_size_for_l0_meta_pin      : 限制 L0 层文件元数据常驻内存大小
Status TableCache::Get(const ReadOptions& options,
                       const InternalKeyComparator& internal_comparator,
                       const FileMetaData& file_meta, const Slice& k,
                       GetContext* get_context,
                       const MutableCFOptions& mutable_cf_options,
                       HistogramImpl* file_read_hist, bool skip_filters,
                       int level, size_t max_file_size_for_l0_meta_pin) {
  auto& fd = file_meta.fd;
  std::string* row_cache_entry = nullptr;
  bool done = false;
  IterKey row_cache_key;
  std::string row_cache_entry_buffer;

  Status s;
  if (ioptions_.row_cache && !get_context->NeedToReadSequence()) {
    // row cahce 存储数据的最新版本，不包含历史版本
    auto user_key = ExtractUserKey(k);
    // 构造 Row Cache Key（[row_cache_id][fd_number][cache_entry_seq_no]）
    uint64_t cache_entry_seq_no =
        CreateRowCacheKeyPrefix(options, fd, k, get_context, row_cache_key);
    // 从 raw cache 读取数据
    done = GetFromRowCache(user_key, row_cache_key, row_cache_key.Size(),
                           get_context, &s, cache_entry_seq_no);
    if (!done) {
      row_cache_entry = &row_cache_entry_buffer;
    }
  }
  TableReader* t = fd.table_reader;
  TypedHandle* handle = nullptr;
  if (s.ok() && !done) {
    // row cache 未命中则开始查找 Table cache
    if (t == nullptr) {
      // 调用 FindTable 查找 SST 文件的元数据
      s = FindTable(options, file_options_, internal_comparator, file_meta,
                    &handle, mutable_cf_options,
                    options.read_tier == kBlockCacheTier /* no_io */,
                    file_read_hist, skip_filters, level,
                    true /* prefetch_index_and_filter_in_cache */,
                    max_file_size_for_l0_meta_pin, file_meta.temperature);
      if (s.ok()) {
        t = cache_.Value(handle);
      }
    }
    SequenceNumber* max_covering_tombstone_seq =
        get_context->max_covering_tombstone_seq();
    if (s.ok() && max_covering_tombstone_seq != nullptr &&
        !options.ignore_range_deletions) {
      // 创建 Range Tombstone 迭代器
      std::unique_ptr<FragmentedRangeTombstoneIterator> range_del_iter(
          t->NewRangeTombstoneIterator(options));
      if (range_del_iter != nullptr) {
        // 检查 Key 是否被 Tombstone 覆盖
        SequenceNumber seq =
            range_del_iter->MaxCoveringTombstoneSeqnum(ExtractUserKey(k));
        if (seq > *max_covering_tombstone_seq) {
          // 如果覆盖当前 Key 的最大 Tombstone 序列号大于 max_covering_tombstone_seq 则更新
          // 以便后续若发现 Key 的序列号小于此值，说明 Key 已被删除，提前终止查询
          *max_covering_tombstone_seq = seq;
          if (get_context->NeedTimestamp()) {
            get_context->SetTimestampFromRangeTombstone(
                range_del_iter->timestamp());
          }
        }
      }
    }
    if (s.ok()) {
      // 后续复用 Get 得到的值 insert 到 raw cache 中，无需重复查询
      get_context->SetReplayLog(row_cache_entry);  // nullptr if no cache.
      // 执行 SST 文件查询
      s = t->Get(options, k, get_context,
                 mutable_cf_options.prefix_extractor.get(), skip_filters);
      get_context->SetReplayLog(nullptr);
    } else if (options.read_tier == kBlockCacheTier && s.IsIncomplete()) {
      // 如果仅支持内存模式，标记 key 可能存在
      get_context->MarkKeyMayExist();
      done = true;
    }
  }

  if (!done && s.ok() && row_cache_entry && !row_cache_entry->empty()) {
    RowCacheInterface row_cache{ioptions_.row_cache.get()};
    size_t charge = row_cache_entry->capacity() + sizeof(std::string);
    auto row_ptr = new std::string(std::move(*row_cache_entry));
    // 数据插入 row cache
    Status rcs = row_cache.Insert(row_cache_key.GetUserKey(), row_ptr, charge);
    if (!rcs.ok()) {
      delete row_ptr;
    }
  }

  if (handle != nullptr) {
    cache_.Release(handle);
  }
  return s;
}

// 从 row cache 查询缓存
bool TableCache::GetFromRowCache(const Slice& user_key, IterKey& row_cache_key,
                                 size_t prefix_size, GetContext* get_context,
                                 Status* read_status, SequenceNumber seq_no) {
  bool found = false;
  // 生成 key
  row_cache_key.TrimAppend(prefix_size, user_key.data(), user_key.size());
  RowCacheInterface row_cache{ioptions_.row_cache.get()};
  if (auto row_handle = row_cache.Lookup(row_cache_key.GetUserKey())) {
    // 通过 cleanable 实现自动释放资源避免泄漏，读流程中有详细介绍
    Cleanable value_pinner;
    // 将 row_handle 的 Release() 操作注册清理任务
    row_cache.RegisterReleaseAsCleanup(row_handle, value_pinner);
    // 解析 cache 缓存的序列化结果并放到 get_context 里
    *read_status = replayGetContextLog(*row_cache.Value(row_handle), user_key,
                                       get_context, &value_pinner, seq_no);
    found = true;
  } 
  return found;
}


// ReadOptions                        : 本次读取的策略
// file_options                       : SST 文件的 I/O 行为
// internal_comparator                : internal_key 的比较逻辑
// file_meta                          : SST 文件的元数据
// no_io                              : 是否禁止磁盘 I/O
// skip_filters                       : 是否跳过加载布隆过滤器
// level                              : 标识 SST 文件所存在的 LSM 层级
// prefetch_index_and_filter_in_cache : 是否预取索引和布隆过滤器到 BlockCache
// max_file_size_for_l0_meta_pin      : 限制 L0 层文件元数据常驻内存大小
// file_temperature                   : 标识文件数据冷热
Status TableCache::FindTable(
    const ReadOptions& ro, const FileOptions& file_options,
    const InternalKeyComparator& internal_comparator,
    const FileMetaData& file_meta, TypedHandle** handle,
    const MutableCFOptions& mutable_cf_options, const bool no_io,
    HistogramImpl* file_read_hist, bool skip_filters, int level,
    bool prefetch_index_and_filter_in_cache,
    size_t max_file_size_for_l0_meta_pin, Temperature file_temperature) {

  // 提取文件的编号作为 LRU 缓存的键
  uint64_t number = file_meta.fd.GetNumber();
  Slice key = GetSliceForFileNumber(&number);
  // 首次查找缓存
  *handle = cache_.Lookup(key);

  if (*handle == nullptr) {
    if (no_io) {
      // 避免触发磁盘操作，避免干扰其他线程的 I/O 操作。
      return Status::Incomplete("Table not found in table_cache, no_io is set");
    }
    // 加锁（按 key 的哈希分片）二次检查
    MutexLock load_lock(&loader_mutex_.Get(key));
    *handle = cache_.Lookup(key);
    if (*handle != nullptr) {
      return Status::OK();
    }

    // 调用 GetTableReader 加载 SST 文件
    std::unique_ptr<TableReader> table_reader;
    Status s = GetTableReader(ro, file_options, internal_comparator, file_meta,
                              false /* sequential mode */, file_read_hist,
                              &table_reader, mutable_cf_options, skip_filters,
                              level, prefetch_index_and_filter_in_cache,
                              max_file_size_for_l0_meta_pin, file_temperature);
    if (!s.ok()) {
      assert(table_reader == nullptr);
    } else {
      // 将 table_reader 插入缓存，并把所有权转移给缓存，通过 handle 进行后续的访问
      s = cache_.Insert(key, table_reader.get(), 1, handle);
      if (s.ok()) {
        table_reader.release();
      }
    }
    return s;
  }
  return Status::OK();
}

Status TableCache::GetTableReader(
    const ReadOptions& ro, const FileOptions& file_options,
    const InternalKeyComparator& internal_comparator,
    const FileMetaData& file_meta, bool sequential_mode,
    HistogramImpl* file_read_hist, std::unique_ptr<TableReader>* table_reader,
    const MutableCFOptions& mutable_cf_options, bool skip_filters, int level,
    bool prefetch_index_and_filter_in_cache,
    size_t max_file_size_for_l0_meta_pin, Temperature file_temperature) {
  // 生成 SST 文件路径
  std::string fname = TableFileName(
      ioptions_.cf_paths, file_meta.fd.GetNumber(), file_meta.fd.GetPathId());
  
  // 创建 FSRandomAccessFile 进行文件读取
  std::unique_ptr<FSRandomAccessFile> file;
  FileOptions fopts = file_options;
  fopts.temperature = file_temperature;
  Status s = PrepareIOFromReadOptions(ro, ioptions_.clock, fopts.io_options);
  if (s.ok()) {
    s = ioptions_.fs->NewRandomAccessFile(fname, fopts, &file, nullptr);
  }
  if (s.ok()) {
    RecordTick(ioptions_.stats, NO_FILE_OPENS);
  } else if (s.IsPathNotFound()) {
    // 如果文件不存在则将 rocksdb 格式替换为 leveldb 进行兼容
    fname = Rocks2LevelTableFileName(fname);
    Status temp_s =
        PrepareIOFromReadOptions(ro, ioptions_.clock, fopts.io_options);
    if (temp_s.ok()) {
      temp_s = ioptions_.fs->NewRandomAccessFile(fname, file_options, &file,
                                                 nullptr);
    }
    if (temp_s.ok()) {
      s = temp_s;
    }
  }

  if (s.ok()) { 
    if (!sequential_mode && ioptions_.advise_random_on_open) {
      // 非顺序扫描通知操作系统尽职 prefetch 减少读放大
      file->Hint(FSRandomAccessFile::kRandom);
    }
    if (ioptions_.default_temperature != Temperature::kUnknown &&
        file_temperature == Temperature::kUnknown) {
      // 冷热数据分层处理
      file_temperature = ioptions_.default_temperature;
    }

    std::unique_ptr<RandomAccessFileReader> file_reader(
        new RandomAccessFileReader(std::move(file), fname, ioptions_.clock,
                                   io_tracer_, ioptions_.stats, SST_READ_MICROS,
                                   file_read_hist, ioptions_.rate_limiter.get(),
                                   ioptions_.listeners, file_temperature,
                                   level == ioptions_.num_levels - 1));
    
    // SST 文件校验
    UniqueId64x2 expected_unique_id;
    if (ioptions_.verify_sst_unique_id_in_manifest) {
      expected_unique_id = file_meta.unique_id;
    } else {
      expected_unique_id = kNullUniqueId64x2;  // null ID == no verification
    }
    // TableReader 创建，元数据加载
    s = mutable_cf_options.table_factory->NewTableReader(
        ro,
        TableReaderOptions(
            ioptions_, mutable_cf_options.prefix_extractor, file_options,
            internal_comparator,
            mutable_cf_options.block_protection_bytes_per_key, skip_filters,
            immortal_tables_, false /* force_direct_prefetch */, level,
            block_cache_tracer_, max_file_size_for_l0_meta_pin, db_session_id_,
            file_meta.fd.GetNumber(), expected_unique_id,
            file_meta.fd.largest_seqno, file_meta.tail_size,
            file_meta.user_defined_timestamps_persisted),
        std::move(file_reader), file_meta.fd.GetFileSize(), table_reader,
        prefetch_index_and_filter_in_cache);
  }
  return s;
}

```

