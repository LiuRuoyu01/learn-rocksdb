## MANIFEST文件

MANIFEST 日志文件是元数据文件，描述了所有列族中 __LSM 树结构信息__的文件，包含 RocksDB 的**状态快照**（每一层sst文件概要信息）和**状态变更记录**，可以通过 MANIFEST 观察当前 LSM 树的结构

CURRENT 文件是一个特殊的指针文件，始终指向最新的 MANIFEST 日志文件

系统（重新）启动时，RocksDB 通过 CURRENT 文件找到最新的 MANIFEST 文件，读取最新的 MANIFEST 文件，加载 RocksDB 的元信息，如果有未完成的状态变更（例如由于系统崩溃导致的未完成操作），通过回放 MANIFEST 中的变更日志恢复数据库的一致性

``` 
./ldb manifest_dump --path="my_rocksdb/MANIFEST-000032"

仅包含 RocksDB 的当前状态快照 
打印如下：
--------------- Column family "default"  (ID 0) --------------   
log number: 10                            //正在使用WAL文件的编号
comparator: leveldb.BytewiseComparator    //RocksDB默认比较器，按照字符大小比较键值
--- level 0 --- version# 1 ---  
// SST文件编号，SST文件大小，SST文件的序号，  SST文件的最小key和最大key
   10:        13759       [2049 .. 3072] ['key0' seq:2049, type:0 .. 'key999' seq:3048, type:0]  
   7:         1076837     [1025 .. 2048] ['key0' seq:1025, type:1 .. 'key999' seq:2024, type:1]     
   4:         1076837     [1 .. 1024]    ['key0' seq:1, type:1 .. 'key999' seq:1000, type:1]    
--- level 1 --- version# 1 ---    
--- level 2 --- version# 1 ---    
..... 省略    
--- level 62 --- version# 1 ---    
--- level 63 --- version# 1 ---
next_file_number 46         //表示下一个sst文件可用的编号为46
last_sequence 3072          //表示上次的写操作的序列号为3072
prev_log_number 0           //表示当前WAL文件之前的一个WAL文件的编号，确保日志文件的顺序和一致性
max_column_family 0         //最大的列族编号，这里是0（只有一个默认列族）
min_log_number_to_keep 0    //2PC模式下使用，恢复过程中忽略小于等于该值的日志

```

##### MANIFEST日志文件生成流程

1. MANIFEST 日志文件会将 RocksDB 的**状态变化（如 SST 文件的创建、删除、合并等操作）** 记录下来。

2. 当 MANIFEST 日志文件大小超过一定阀值时，RocksDB 会创建一个新的 MANIFEST 日志文件，新的 MANIFEST 日志文件只会记录当前的**数据库状态（例如 SST 文件列表、层级信息、键范围等）快照**，更新 CURRENT 文件，使其指向新的 MANIFEST 文件。

3. 之后 RocksDB 会删除冗余的旧 MANIFEST 文件，释放存储空间。

##### MANIFEST记录SST文件的动态信息（状态变化）

- MANIFEST 文件是追加写日志文件，采用 **log entry** 的形式，每次 SST 文件的变更会以新记录的形式追加到 MANIFEST 中

### 问题

在数据库恢复过程中，RocksDB 通过回放 **MANIFEST 文件** 来确定磁盘上仍然存活的内容，为了恢复内存中未刷写到磁盘的内容，RocksDB 依赖于列出磁盘上的 **WAL（日志文件）**，并按顺序回放它们，但是如果某个 WAL 文件丢失或损坏，RocksDB 无法察觉这一点，并会在未检测到错误的情况下进行恢复，最终导致恢复出的内容存在**数据损坏**的问题

1. 如果 **WAL 目录（元数据）未进行同步**，在系统崩溃时，WAL 文件可能**并未真正存在于磁盘上**。
2. 如果 **WAL 文件在关闭后未被同步**，操作系统通常会异步将数据写入磁盘，如果系统在异步写入完成前宕机，部分数据可能丢失，则恢复时无法可靠检查 WAL 文件的大小是否正确

#### 设计

让 MANIFEST 不仅跟踪 SST 文件，还跟踪 WAL。跟踪 WAL 的意义在于记录其生命周期事件（例如创建、关闭和变为无效）以及元数据（例如日志编号、大小、校验和）。通过在数据库恢复时重放 MANIFEST，可以确定哪些 WAL 是有效的。

1. 从 MANIFEST 中恢复 WAL 的跟踪信息并将其存储在 VersionSet 中。
2. 从 VersionSet 中获取 WAL 。
3. 按日志编号的递增顺序对每个 WAL 进行检查
   1. 如果 WAL 未同步，则跳过检查。
   2. 如果磁盘上缺少 WAL，或者 WAL 的大小小于上次同步的大小，则报告错误。

##### MANIFEST 记录 WAL 同步事件的方案

1. 每次同步记录当前 WAL 大小。（如果写入频率高会导致写操作变慢并且 MANIFEST 文件中充斥大量 WAL 相关记录）
2. 仅记录 WAL 是否被同步。（无法确切知道 WAL 的同步大小）
3. 内存中记录同步的大小，当创建新的 WAL 时，将上一个 WAL 的最后同步大小写入 MANIFSEST。当调用写入的命令（如 SyncWAL 或 FlushWAL(true)），所有未完全同步的存活 WAL 将被完全同步，并将这些 WAL 的大小写入 MANIFEST。（对于当前活跃的 WAL，其同步大小不会被写入 MANIFEST）

##### MANIFEST 跟踪WAL 生命周期

WAL 文件被[重用](https://github.com/LiuRuoyu01/learn-rocksdb/blob/main/ch02/RocksDB_WAL.md#io%E6%95%B0%E9%87%8F)时，他会被逻辑上记为删除，因此也会在 MANIFEST 文件记录一个删除事件（和非重用文件流程完全相同）

1. WAL 创建时：将日志编号写入 MANIFEST，以指示已创建新的 WAL。

2. WAL 完成写入时：

   - 如果WAL已经完成写入并同步，则将日志编号和WAL的一些完整性信息（如大小，校验和）写入MANIFEST，以表明WAL已完成。

   -  如果 WAL 已经完成写入但未同步，则不要将 WAL 的大小或校验和写入 MANIFEST。在这种情况下，从 MANIFEST 的角度来看，它与 WAL 创建相同，因为在恢复时，我们只知道 WAL 的日志编号，但不知道其大小或校验和

3. WAL 过时：

   追踪过时的方式

   1. **追踪 WAL 的过时事件**：通过在 MANIFEST 中记录 WAL 的过时事件，确保 WAL 的生命周期（包括创建、关闭、过时）都被跟踪，可以准确地知道哪些 WAL 文件仍然存活，哪些已经过时。MANIFEST 中与 WAL 相关的 VersionEdits 可以作为哪些 WAL 处于活动状态的真实来源。
   2. **通过** min_log_number **确定 WAL 是否过时**：通过持久化空列族的最新日志编号，从而确保 min_log_number 是准确的，并能确保 WAL 的过时状态
      - 由于不同列族共享 WAL 日志，如果有 N 个这样的列族，那么每次创建新的 WAL 时，我们都需要向 MANIFEST 写入 N 个 VersionEdits，以将当前日志编号记录为空列族的 min_log_number。（比如列族 1 很久没有更新，列族 2 一直在更新，并且不断产生新的 WAL 文件。对于列族 1，它长时间没有进行任何更新，所以列族 1 的 log_number 可能会被错误地认为是较小的，导致恢复时认为某些不再需要的 WAL 文件依然是有效的。为了确保其 log_number 不会被忽略，因此它的 log_number 会被设置为**最大值**，通常是一个非常大的数字，并且这个信息没有持久化到 MANIFEST 文件中，RocksDB 会在 MANIFEST 中**写入一个空的 VersionEdit**，表示列族 1 的最新 log_number）

   追踪过时的时间：

   1. **物理删除时追踪（废弃）**
      - 异步删除 WAL 可能 MANIFEST 中未更新
      - 在崩溃恢复时，如果过时事件已经写入，但 WAL 文件未物理删除，系统会发现 WAL 文件在磁盘上。
   2. **逻辑删除时追踪（默认）**
      -  MANIFEST 文件中可以精确地记录哪些 WAL 文件已经不再活跃，并且有了 WAL 文件的生命周期信息
      - 在刷新 memtable 后，相关的 VersionEdit 可以一起写入，这样避免了额外的磁盘 I/O 操作

##### WAL 相关设计

- 采用了WalAddition 和 WalDeletion，其中
  - WalAddition：表示 WAL 文件的创建和写入完成；

  - WalDeletion：表示 WAL 文件的删除。

- 每个 VersionEdit 要么是 WalAddition，要么是 WalDeletion，不能同时包含两者
- LogAndApply 中对 WAL 相关 VersionEdit 的特殊处理 ：在 VersionSet 中维护一个独立的数据结构，用于跟踪存活的 WAL 文件，每次 WAL 文件的变更都通过 WalAddition 和 WalDeletion 更新这个独立数据结构，而不是创建新的列族版本

```c++
// column_family_datas：           需要进行操作的所有列族。
// mutable_cf_options_list：       每个列族对应的可变列族选项。
// read_options 和 write_options： 用于指定读取和写入操作的选项
// edit_lists：                    包含了对多个列族的一系列修改
// mu：                            互斥锁
// dir_contains_current_file：     当前 MANIFEST 文件是否包含在指定目录中的指针
// new_descriptor_log：            指示是否需要切换到新的 MANIFEST 文件。
// new_cf_options：                针对新的列族选项。
// manifest_wcbs：                 回调函数列表
Status VersionSet::LogAndApply(
    const autovector<ColumnFamilyData*>& column_family_datas,
    const autovector<const MutableCFOptions*>& mutable_cf_options_list,
    const ReadOptions& read_options, const WriteOptions& write_options,
    const autovector<autovector<VersionEdit*>>& edit_lists,
    InstrumentedMutex* mu, FSDirectory* dir_contains_current_file,
    bool new_descriptor_log, const ColumnFamilyOptions* new_cf_options,
    const std::vector<std::function<void(const Status&)>>& manifest_wcbs) {
  mu->AssertHeld();
  // 计算所有的操作数量
  int num_edits = 0;
  for (const auto& elist : edit_lists) {
    num_edits += static_cast<int>(elist.size());
  }

  // 针对每个列族创建 ManifestWriter
  int num_cfds = static_cast<int>(column_family_datas.size());
  std::deque<ManifestWriter> writers;
  for (int i = 0; i < num_cfds; ++i) {
    const auto wcb =
        manifest_wcbs.empty() ? [](const Status&) {} : manifest_wcbs[i];
    // ManifestWriter 分别加入到局部队列和全局队列
    writers.emplace_back(mu, column_family_datas[i],
                         *mutable_cf_options_list[i], edit_lists[i], wcb);
    manifest_writers_.push_back(&writers[i]);
  }
  
  // 保证顺序执行
  // 只有当之前保存在全局队列中的所有 Writers 都写入完毕之后才会执行此次局部队列，否则就会等待 
  ManifestWriter& first_writer = writers.front();
  while (!first_writer.done && &first_writer != manifest_writers_.front()) {
    first_writer.cv.Wait();
  }
  if (first_writer.done) {
    return first_writer.status;
  }

  // 统计没有被删除的列族数量
  int num_undropped_cfds = 0;
  for (auto cfd : column_family_datas) {
    if (cfd == nullptr || !cfd->IsDropped()) {
      ++num_undropped_cfds;
    }
  }
  
  // 如果所有列族都被删除，则所有的写操作都会被丢弃
  if (0 == num_undropped_cfds) {
    for (int i = 0; i != num_cfds; ++i) {
      manifest_writers_.pop_front();
    }
    if (!manifest_writers_.empty()) {
      manifest_writers_.front()->cv.Signal();
    }
    return Status::ColumnFamilyDropped();
  }
  
  return ProcessManifestWrites(writers, mu, dir_contains_current_file,
                               new_descriptor_log, new_cf_options, read_options,
                               write_options);
}


// writers:                       包含所有待处理的 ManifestWriter。
// mu:                            数据库锁，保证线程安全。
// dir_contains_current_file:     表示包含当前文件的目录。
// new_descriptor_log:            指示是否需要切换到新的 MANIFEST 文件。
// new_cf_options:                针对新列族的配置选项。
// read_options 和 write_options: 读写操作的配置参数。
Status VersionSet::ProcessManifestWrites(
    std::deque<ManifestWriter>& writers, InstrumentedMutex* mu,
    FSDirectory* dir_contains_current_file, bool new_descriptor_log,
    const ColumnFamilyOptions* new_cf_options, const ReadOptions& read_options,
    const WriteOptions& write_options) {
  
  // 取出 writers 队列中的第一个 ManifestWriter 作为当前的写操作
  ManifestWriter& first_writer = writers.front();
  ManifestWriter* last_writer = &first_writer;

  // ...

  if (first_writer.edit_list.front()->IsColumnFamilyManipulation()) {
    // 如果操作是列族的添加或删除结构变化，直接调用 LogAndApplyCFHelper
    LogAndApplyCFHelper(first_writer.edit_list.front(), &max_last_sequence);
    batch_edits.push_back(first_writer.edit_list.front());
    batch_edits_ts_sz.push_back(std::nullopt);
  } else {
    auto it = manifest_writers_.cbegin();
    // 记录组的起始位置，写操作时是事务，只能全成功或全失败
    size_t group_start = std::numeric_limits<size_t>::max();
    while (it != manifest_writers_.cend()) {
      // 列族的添加或删除结构变化需要单独处理
      if ((*it)->edit_list.front()->IsColumnFamilyManipulation()) {
        break;
      }
      
      // 获取当前的 ManifestWriter
      last_writer = *(it++);
      
      // 特殊处理已删除的列族
      if (last_writer->cfd->IsDropped()) {
        if (!batch_edits.empty()) {
          if (batch_edits.back()->IsInAtomicGroup() &&
              batch_edits.back()->GetRemainingEntries() > 0) {
            // 仅针对原子组（事务），并且还有操作未处理完
            const auto& edit_list = last_writer->edit_list;
            size_t k = 0;
            while (k < edit_list.size()) {
              if (!edit_list[k]->IsInAtomicGroup()) {
                break;
              } else if (edit_list[k]->GetRemainingEntries() == 0) {
                ++k;
                break;
              }
              ++k;
            }
            // 当列族被删除时，后续与该列族相关的操作会被跳过，不能写入到 MANIFEST 文件
            // 如果原子组中有剩余的操作没有执行，通过更新 remaining_entries_ 来反映
            // 已经跳过的操作数，避免恢复时系统认为有些未执行的操作依据完成出现错误
            for (auto i = ; i < batch_edits.size(); ++i) {
              batch_edits[i]->SetRemainingEntries(
                  batch_edits[i]->GetRemainingEntries() -
                  static_cast<uint32_t>(k));
            }
          }
        }
        continue;
      }
      
      // 为每个列族找到或创建对应的 Version 和 VersionBuilder
      Version* version = nullptr;
      VersionBuilder* builder = nullptr;
      for (int i = 0; i != static_cast<int>(versions.size()); ++i) {
        uint32_t cf_id = last_writer->cfd->GetID();
        if (versions[i]->cfd()->GetID() == cf_id) {
          version = versions[i];
          builder = builder_guards[i]->version_builder();
          break;
        }
      }
      if (version == nullptr) {
        if (!last_writer->IsAllWalEdits()) {
          // wal 不需要应用到 version
          // 为列族创建一个新的 Version 对象
          // 及其关联的 VersionBuilder，用于后续的版本管理和变更操作
          version = new Version(last_writer->cfd, this, file_options_,
                                last_writer->mutable_cf_options, io_tracer_,
                                current_version_number_++);
          versions.push_back(version);
          mutable_cf_options_ptrs.push_back(&last_writer->mutable_cf_options);
          builder_guards.emplace_back(
              new BaseReferencedVersionBuilder(last_writer->cfd));
          builder = builder_guards.back()->version_builder();
        }
      }
      for (const auto& e : last_writer->edit_list) {
        if (e->IsInAtomicGroup()) {
          // 标记原子组的起始位置
          if (batch_edits.empty() || !batch_edits.back()->IsInAtomicGroup() ||
              (batch_edits.back()->IsInAtomicGroup() &&
               batch_edits.back()->GetRemainingEntries() == 0)) {
            group_start = batch_edits.size();
          }
        } else if (group_start != std::numeric_limits<size_t>::max()) {
          // 无效值
          group_start = std::numeric_limits<size_t>::max();
        }
        // 将 VersionEdit 应用到 VersionBuilder 上
        Status s = LogAndApplyHelper(last_writer->cfd, builder, e,
                                     &max_last_sequence, mu);
        
        // 如果错误释放已分配的资源...
        
        batch_edits.push_back(e);
        batch_edits_ts_sz.push_back(edit_ts_sz);
      }
    } 
    
    for (int i = 0; i < static_cast<int>(versions.size()); ++i) {
      // 将版本信息保存在对应的位置
      auto* builder = builder_guards[i]->version_builder();
      Status s = builder->SaveTo(versions[i]->storage_info());
      // 如果错误释放已分配的资源...
    }
  }
  
  if (!descriptor_log_ ||
      manifest_file_size_ > db_options_->max_manifest_file_size) {
    // 如果 MANIFEST 尚未初始化或者当前文件的大小超过限制，需要创建新文件
    new_descriptor_log = true;
  } else {
    // 暂存当前的 MANIFEST 文件编号
    pending_manifest_file_number_ = manifest_file_number_;
  }

  // 缓存CF状态，CF_Id -> CF_State
  std::unordered_map<uint32_t, MutableCFState> curr_state;
  VersionEdit wal_additions;
  if (new_descriptor_log) {
    // 设置文件编号
    pending_manifest_file_number_ = NewFileNumber();
    batch_edits.back()->SetNextFile(next_file_number_.load());
    // 确保 MANIFSEST 包含最大列族 ID
    if (column_family_set_->GetMaxColumnFamily() > 0) {
      first_writer.edit_list.front()->SetMaxColumnFamily(
          column_family_set_->GetMaxColumnFamily());
    }
    // 记录状态
    for (const auto* cfd : *column_family_set_) {
      curr_state.emplace(
          cfd->GetID(),
          MutableCFState(cfd->GetLogNumber(), cfd->GetFullHistoryTsLow()));
    }
    // 添加 WAL 文件的记录		
    for (const auto& wal : wals_.GetWals()) {
      wal_additions.AddWal(wal.first, wal.second);
    }
  }
  
  {
    // 写入新的 MANIFEST 文件之前的准备工作 ...

    log::Writer* raw_desc_log_ptr = descriptor_log_.get()；
    if (s.ok() && new_descriptor_log) {
      // 创建新的 MANIFEST
      std::string descriptor_fname =
          DescriptorFileName(dbname_, pending_manifest_file_number_);
      std::unique_ptr<FSWritableFile> descriptor_file;
      io_s = NewWritableFile(fs_.get(), descriptor_fname, &descriptor_file,
                             opt_file_opts);
      if (io_s.ok()) {
        // 设置文件的预分配大小
        descriptor_file->SetPreallocationBlockSize(
            db_options_->manifest_preallocation_size);
        
        // ...

        // 写入当前记录到 manifest
        s = WriteCurrentStateToManifest(write_options, curr_state,
                                        wal_additions, raw_desc_log_ptr, io_s);
      }
    }

    if (s.ok()) {
      if (!first_writer.edit_list.front()->IsColumnFamilyManipulation()) {
        constexpr bool update_stats = true;
        for (int i = 0; i < static_cast<int>(versions.size()); ++i) {
          // 在非列族结构操作时，确保版本对象的元数据准备完毕
          versions[i]->PrepareAppend(*mutable_cf_options_ptrs[i], read_options,
                                     update_stats);
        }
      }
      // 写入新纪录到日志
      for (size_t bidx = 0; bidx < batch_edits.size(); bidx++) {
        auto& e = batch_edits[bidx];
        // 将文件添加到隔离列表，确保写入失败不会影响数据库的一致性和恢复
        files_to_quarantine_if_commit_fail.push_back(
            e->GetFilesToQuarantineIfCommitFail());
        // 编码变更日志
        std::string record;
        if (!e->EncodeTo(&record, batch_edits_ts_sz[bidx])) {
          s = Status::Corruption("Unable to encode VersionEdit:" +
                                 e->DebugString(true));
          break;
        }
        // 将编码后的数据写入到 MANIFEST 文件中
        io_s = raw_desc_log_ptr->AddRecord(write_options, record);
        if (!io_s.ok()) {
          s = io_s;
          manifest_io_status = io_s;
          break;
        }
      }

      if (s.ok()) {
        // 将 MANIFEST 文件的内容同步到磁盘
        io_s =
            SyncManifest(db_options_, write_options, raw_desc_log_ptr->file());
        manifest_io_status = io_s;
      }
      if (!io_s.ok()) {
        s = io_s;
      }
    }

    if (s.ok() && new_descriptor_log) {
      // 如果创建了新的 MANIFSEST 文件，更新 CURRENT 指向 MANIFSEST
      io_s = SetCurrentFile(
          write_options, fs_.get(), dbname_, pending_manifest_file_number_,
          file_options_.temperature, dir_contains_current_file);
      if (!io_s.ok()) {
        s = io_s;
        // CURRENT 文件失败，旧的 MANIFEST 文件可能仍然有效，所以需要将其隔离以供恢复时使用
        limbo_descriptor_log_file_number.push_back(manifest_file_number_);
        files_to_quarantine_if_commit_fail.push_back(
            &limbo_descriptor_log_file_number);
      }
    }

    // ...
  }

  if (s.ok()) {
    for (auto& e : batch_edits) {
      // 更新 WAL
      if (e->IsWalAddition()) {
        s = wals_.AddWals(e->GetWalAdditions());
      } else if (e->IsWalDeletion()) {
        s = wals_.DeleteWalsBefore(e->GetWalDeletion().GetLogNumber());
      }
    }
  }

  // ...

  if (s.ok() && new_descriptor_log) {
    descriptor_log_ = std::move(new_desc_log_ptr);
    // 一个存储已废弃的 manifest 文件路径的列表，等待稍后删除
    obsolete_manifests_.emplace_back(
        DescriptorFileName("", manifest_file_number_));
  }

  if (s.ok()) {
    if (first_writer.edit_list.front()->IsColumnFamilyAdd()) {
      // 处理 CF 的增加
      CreateColumnFamily(*new_cf_options, read_options,
                         first_writer.edit_list.front());
    } else if (first_writer.edit_list.front()->IsColumnFamilyDrop()) {
      // 处理 CF 的删除
      first_writer.cfd->SetDropped();
      first_writer.cfd->UnrefAndTryDelete();
    } else {
      // 对于每个列族，更新其日志编号，表示应忽略编号小于该编号的日志。
      uint64_t last_min_log_number_to_keep = 0;
      for (const auto& e : batch_edits) {
        ColumnFamilyData* cfd = nullptr;
        if (!e->IsColumnFamilyManipulation()) {
          cfd = column_family_set_->GetColumnFamily(e->GetColumnFamily());
        }
        // 列族如果被删除不记录操作
        if (cfd) {
          // 更新列族的日志号
          if (e->HasLogNumber() && e->GetLogNumber() > cfd->GetLogNumber()) {
            cfd->SetLogNumber(e->GetLogNumber());
          }
          if (e->HasFullHistoryTsLow()) {
            cfd->SetFullHistoryTsLow(e->GetFullHistoryTsLow());
          }
          if (e->HasMinLogNumberToKeep()) {
          last_min_log_number_to_keep =
              std::max(last_min_log_number_to_keep, e->GetMinLogNumberToKeep());
          }
        }
      } 
      // 更新最小日志号
      if (last_min_log_number_to_keep != 0) {
        MarkMinLogNumberToKeep(last_min_log_number_to_keep);
      }
      // 将 Version 添加到列族中的版本链表
      for (int i = 0; i < static_cast<int>(versions.size()); ++i) {
        ColumnFamilyData* cfd = versions[i]->cfd_;
        AppendVersion(cfd, versions[i]);
      }
    }
    // 更新 Manifest 和日志相关的信息
    descriptor_last_sequence_ = max_last_sequence;
    manifest_file_number_ = pending_manifest_file_number_;
    manifest_file_size_ = new_manifest_file_size;
    prev_log_number_ = first_writer.edit_list.front()->GetPrevLogNumber();
  } else {
    // 处理错误情况
    
    // 如果发生错误导致文件损坏，需要放弃当前的日志对象，避免损坏的日志对象被进一步使用
    // 强制下一个版本更新时开始使用一个新的 MANIFEST 文件
    descriptor_log_.reset();
    new_desc_log_ptr.reset();
    
    if (!manifest_io_status.ok() && new_descriptor_log) {
      // 如果 MANIFEST 操作失败，CURRENT 文件仍然指向原始的 MANIFEST。
    	// 因此，可以安全地删除新的 MANIFEST 文件。
      Status manifest_del_status = env_->DeleteFile(
          DescriptorFileName(dbname_, pending_manifest_file_number_));
    }
    // 如果出现 CURRENT 错误，通常不会删除新的 MANIFEST 文件
  }

  // 唤醒等待的 writers
  while (true) {
    ManifestWriter* ready = manifest_writers_.front();
    manifest_writers_.pop_front();
    bool need_signal = true;
    
    // 遍历 writers，检查当前处理的 ManifestWriter 是否是 writers 中的一
    for (const auto& w : writers) {
      if (&w == ready) {
        need_signal = false;
        break;
      }
    }
    // 设置当前 writer 的状态为写入结果
    ready->status = s;
    ready->done = true;
    if (ready->manifest_write_callback) {
      (ready->manifest_write_callback)(s);
    }
    // 如果该 ManifestWriter 在 writers 中没有找到，则需要发送信号
    if (need_signal) {
      ready->cv.Signal();
    }
    if (ready == last_writer) {
      break;
    }
  }
  // 唤醒下一批 writers 执行
  if (!manifest_writers_.empty()) {
    manifest_writers_.front()->cv.Signal();
  }
  
}

Status VersionSet::WriteCurrentStateToManifest(
    const WriteOptions& write_options,
    const std::unordered_map<uint32_t, MutableCFState>& curr_state,
    const VersionEdit& wal_additions, log::Writer* log, IOStatus& io_s) {

  // 记录新的 WAL 增加记录
  if (!wal_additions.GetWalAdditions().empty()) {
    std::string record;
    if (!wal_additions.EncodeTo(&record)) {
      return Status::Corruption("Unable to Encode VersionEdit: " +
                                wal_additions.DebugString(true));
    }
    io_s = log->AddRecord(write_options, record);
  }
	
  // 记录删除的 WAL 文件
  VersionEdit wal_deletions;
  wal_deletions.DeleteWalsBefore(min_log_number_to_keep());
  std::string wal_deletions_record;
  if (!wal_deletions.EncodeTo(&wal_deletions_record)) {
    return Status::Corruption("Unable to Encode VersionEdit: " +
                              wal_deletions.DebugString(true));
  }
  io_s = log->AddRecord(write_options, wal_deletions_record);

  for (auto cfd : *column_family_set_) {
    if (cfd->IsDropped()) {
      // 已删除的列族不会记录
      continue;
    }
    {
      // 保存列族的元数据
      VersionEdit edit;
      if (cfd->GetID() != 0) {
        edit.AddColumnFamily(cfd->GetName());
        edit.SetColumnFamily(cfd->GetID());
      }
      edit.SetComparatorName(
          cfd->internal_comparator().user_comparator()->Name());
      edit.SetPersistUserDefinedTimestamps(
          cfd->ioptions()->persist_user_defined_timestamps);
      std::string record;
      if (!edit.EncodeTo(&record)) {
        return Status::Corruption("Unable to Encode VersionEdit:" +
                                  edit.DebugString(true));
      }
      io_s = log->AddRecord(write_options, record);
    }

    {
      // 保存列族中 SST 文件的相关元数据
      VersionEdit edit;
      edit.SetColumnFamily(cfd->GetID());
      const auto* current = cfd->current();
      const auto* vstorage = current->storage_info();
      // 遍历列族的每个 Level，记录所有的信息
      for (int level = 0; level < cfd->NumberLevels(); level++) {
        const auto& level_files = vstorage->LevelFiles(level);
        for (const auto& f : level_files) {
          edit.AddFile(level, f->fd.GetNumber(), f->fd.GetPathId(),
                       f->fd.GetFileSize(), f->smallest, f->largest,
                       f->fd.smallest_seqno, f->fd.largest_seqno,
                       f->marked_for_compaction, f->temperature,
                       f->oldest_blob_file_number, f->oldest_ancester_time,
                       f->file_creation_time, f->epoch_number, f->file_checksum,
                       f->file_checksum_func_name, f->unique_id,
                       f->compensated_range_deletion_size, f->tail_size,
                       f->user_defined_timestamps_persisted);
        }
      }
      edit.SetCompactCursors(vstorage->GetCompactCursors());
			
      // Blob 文件是 RocksDB 中的一种存储方式，用于存储大对象，如大文本或大二进制数据
      // 记录 Blob 文件的信息
      const auto& blob_files = vstorage->GetBlobFiles();
      for (const auto& meta : blob_files) {
        const uint64_t blob_file_number = meta->GetBlobFileNumber();
        edit.AddBlobFile(blob_file_number, meta->GetTotalBlobCount(),
                         meta->GetTotalBlobBytes(), meta->GetChecksumMethod(),
                         meta->GetChecksumValue());
        if (meta->GetGarbageBlobCount() > 0) {
          edit.AddBlobFileGarbage(blob_file_number, meta->GetGarbageBlobCount(),
                                  meta->GetGarbageBlobBytes());
        }
      }
      
      // 获取当前列族的日志信息
      const auto iter = curr_state.find(cfd->GetID());
      uint64_t log_number = iter->second.log_number;
      edit.SetLogNumber(log_number);

      // min_log_number_to_keep 是整个数据库的全局属性，表示数据库在恢复时可以忽略的最小日志号
      // 默认列族会记录并保存最小日志号，以便在数据库恢复时能够正确地删除无用的日志文件
      if (cfd->GetID() == 0) {
        uint64_t min_log = min_log_number_to_keep();
        if (min_log != 0) {
          edit.SetMinLogNumberToKeep(min_log);
        }
      }

      // 设置时间戳
      const std::string& full_history_ts_low = iter->second.full_history_ts_low;
      if (!full_history_ts_low.empty()) {
        edit.SetFullHistoryTsLow(full_history_ts_low);
      }
			
      // 设置列族的最后序列号，全局序列号，表示到目前为止所有写入操作的最大序列号
      edit.SetLastSequence(descriptor_last_sequence_);

      // 将 VersionEdit 进行编码并保存
      const Comparator* ucmp = cfd->user_comparator();
      std::string record;
      if (!edit.EncodeTo(&record, ucmp->timestamp_size())) {
        return Status::Corruption("Unable to Encode VersionEdit:" +
                                  edit.DebugString(true));
      }
      io_s = log->AddRecord(write_options, record);
      if (!io_s.ok()) {
        return io_s;
      }
    }
  }
  return Status::OK();
}
```

