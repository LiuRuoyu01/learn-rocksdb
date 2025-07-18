

## WAl文件

RocksDB 的每个更新都会写入两个位置：

1. 内存中的数据结构memtable（稍后将刷新到SST文件）
2. 在磁盘上预写日志（WAL）。

WAL是记录__写操作日志__的文件。它的主要功能是确保在数据库发生故障（如宕机或崩溃）时，可以通过日志恢复未刷入 SST 文件的数据，从而避免数据丢失，**单个 WAL 捕获所有列族的写入日志**。

**WAL 中的所有数据都已持久保存到 SST 文件中时，WAL 将被删除**。如果启用了**archival**，则 WAL 将被移动到单独的位置，稍后将从磁盘中清除。

#### WAL 文件的逻辑结构

``` 
./ldb dump_wal --walfile=my_rocksdb/000031.log --header --print_value
打印如下 
Sequence,Count,ByteSize,Offset,Physical  Key(s) : value     
1,       1,    25,      0,      PUT(0) : 0x6B657932 : 0x76616C756532 
2,       1,    25,      32,     PUT(0) : 0x6B657933 : 0x76616C756534 
以第一行为例：序列号:1，当前操作的个数:1，字节数:25，物理偏移:0，PUT(0)表示写入操作，写入列族id为0，即默认列族，后面分别是key的值和value的值，0x6B657932转为ascii即key2，0x6B657932为value2
```

#### WAL 文件的二进制结构

日志文件由一系列长度可变的记录组成。记录按 kBlockSize(32k) 分组。写入器以块为单位写入，读取器以块为单位读取kBlockSize。

每条记录必须存储在一个或多个块中，记录需要连续的写入块中

- 如果块剩余大小小于等于6字节（只够存放CRC和Size）,则需要将此块的剩余字节填充为零。

- 如果块恰好剩余7字节并且添加了一个非零长度的记录，则必须在此块写入一个Type为First的记录（CRC+Size+Type+零字节数据），然后在后续块填充数据
- 如果如果块剩余大小大于7字节，则必须在此块后面写入新纪录

```
+---------+-----------+-----------+--- ... ---+
|CRC (4B) | Size (2B) | Type (1B) | Payload   |
+---------+-----------+-----------+--- ... ---+
CRC:         校验和，用于验证日志记录的完整性，确保日志没有损坏。
Size:        一条记录的总大小（包括头部和负载部分)
Type:        记录类型，用于区分操作类别
Payload:     实际数据部分

Type:
FULL == 1       // 包含整个用户记录的内容
FIRST == 2			// 用户记录第一个片段的类型
MIDDLE == 3     // 用户记录最后一个片段的类型
LAST == 4       // 用户记录所有内部片段的类型
```

优点：

1. 块之间是独立的，如果某一块发生损坏，可以直接跳过这一块，开始从下一个块读取
2. 分块式格式的日志文件记录以块为单位，记录分片的边界总是对齐到块的起始位置，每个块的记录都是完整的，跨块的记录会被明确分片（通过 FIRST、MIDDLE、LAST 类型标记）。
3. 如果一个记录过大，无法完全存储在一个块中，分块日志格式会将记录拆分为多个部分，写入时，直接按顺序将数据写入块，无需在内存中为完整记录预先分配缓冲区

缺点：

1. 每条记录都会有元数据，如果数据量小会导致元数据开销变大。如果支持将多个小记录打包到一个块中，可以显著提高存储效率。



#### WAL 写入模式

##### 异步模式

- 数据写入不会立即写磁盘，只会写入操作系统的页缓存，操作系统因某些原因（如脏页过多）触发刷新，数据才会写入磁盘，用户无需等待I/O，写入操作延迟低，此模式不具备崩溃安全

- 优化：Options.manual_wal_flush = true，WAL直接保存在RocksDB得内部缓冲区，调用 DB::FlushWAL() 手动将缓冲的 WAL 条目写入文件系统，进一步减少写入 OS 页缓存的 CPU 开销，此模式同样不具备崩溃安全

##### 同步模式

- 每次写入操作返回前，都会对 WAL 文件执行 fsync，确保数据持久化到磁盘，提供崩溃安全性。写入延迟会显著增加，需要等待 I/O 完成。

##### group commit

- 当多个线程同时向同一个 DB 写入时，RocksDB 会将符合条件的写入请求合并为一个 batch 进行 WAL 写入，并执行一次 fsync。不同的写选项可能导致请求无法合并（例如同步与异步写入混合时），group commit 的最大为 1MB。

##### Recycle

1. 默认情况下（**Options.recycle_log_file_num = false**），每次创建新的 WAL 文件，调用 fsync 会修改文件的数据和大小，因此需要至少两次 I/O（数据和元数据），虽然 RocksDB 使用 fallocate() 预分配文件空间，但这无法完全避免元数据 I/O

2. 启用 WAL 文件 recycle 模式（**Options.recycle_log_file_num = true**），WAL 文件不会被删除，而是被 recycle，从文件的开始位置随机写入，直接覆盖旧数据，初始阶段文件大小不变，直到写入量超过原文件大小。若写入未超过原文件大小，仅需要更新数据信息，因此可以避免元数据 I/O。如果WAL 文件大小相似，元数据 I/O 开销将被最小化

###### 写放大

- 当写入数据量较小时，文件系统也需要更新整个页，以确保数据完整性，这意味着小数据量占用了整页的空间

- 文件系统还需要更新元数据信息，这些元数据信息也需要写会磁盘，也需要占用整页的空间



#### 写入 WAL

```c++
// 单线程写入 WAL
IOStatus DBImpl::WriteToWAL(const WriteThread::WriteGroup& write_group,
                            log::Writer* log_writer, uint64_t* log_used,
                            bool need_log_sync, bool need_log_dir_sync,
                            SequenceNumber sequence,
                            LogFileNumberSize& log_file_number_size) {
  IOStatus io_s;
  assert(!two_write_queues_);
  assert(!write_group.leader->disable_wal);
  // 合并事务组中的多个 WriteBatch，提高 WAL 写入效率
  size_t write_with_wal = 0;
  WriteBatch* to_be_cached_state = nullptr;
  WriteBatch* merged_batch;
  io_s = status_to_io_status(MergeBatch(write_group, &tmp_batch_, &merged_batch,
                                        &write_with_wal, &to_be_cached_state));
  
  if (merged_batch == write_group.leader->batch) {
    // Leader 线程的 log_used 记录当前 logfile_number_
    write_group.leader->log_used = logfile_number_;
  } else if (write_with_wal > 1) {
    // 如果多个事务写入 WAL，所有 write_group 的事务共享相同的 logfile_number_
    for (auto writer : write_group) {
      writer->log_used = logfile_number_;
    }
  }

  WriteBatchInternal::SetSequence(merged_batch, sequence);

  // 真正执行 WAL 写入
  uint64_t log_size;
  WriteOptions write_options;
  write_options.rate_limiter_priority =
      write_group.leader->rate_limiter_priority;
  io_s = WriteToWAL(*merged_batch, write_options, log_writer, log_used,
                    &log_size, log_file_number_size);

  if (io_s.ok() && need_log_sync) {
    // 在 Sync（）操作前，不加锁读取 logs_
    //  - getting_synced=true，标志意味着当前日志正在进行同步，因此其他线程不会尝试删除 logs_ 中的日志文件
    //  - 只有写线程能够向 logs_ 添加新的 WAL 文件，
    //  - 只要其他线程不修改 logs_，多个线程并发读 std::deque 安全
    
    // Sync() 操作时需要加锁
    //  manual_wal_flush_ 允许手动触发其他线程 FlushWAL
    //  在 WAL 同步到磁盘期间，如果多个线程同时 Sync()，可能会出现文件损坏的情况。
    const bool needs_locking = manual_wal_flush_ && !two_write_queues_;
    if (UNLIKELY(needs_locking)) {
      log_write_mutex_.Lock();
    }

    // 进行 WAL 同步
    if (io_s.ok()) {
      for (auto& log : logs_) {
        IOOptions opts;
        io_s = WritableFileWriter::PrepareIOOptions(write_options, opts);
        if (!io_s.ok()) {
          break;
        }
        // 如果之前的 WAL 文件同步操作失败，可能已经完全同步且已关闭
        // 只需要在 Manifest 中标记为已同步
        if (auto* f = log.writer->file()) {
          io_s = f->Sync(opts, immutable_db_options_.use_fsync);
          if (!io_s.ok()) {
            break;
          }
        }
      }
    }

    if (UNLIKELY(needs_locking)) {
      log_write_mutex_.Unlock();
    }

    // 进行 WAL 目录同步
    if (io_s.ok() && need_log_dir_sync) {
      // WAL 目录只会在第一次 WAL 同步时进行同步，如果用户未启用 WAL 同步也不会对目录进行同步
      io_s = directories_.GetWalDir()->FsyncWithDirOptions(
          IOOptions(), nullptr,
          DirFsyncOptions(DirFsyncOptions::FsyncReason::kNewFileSynced));
    }
  }
  // ...
}

// 在 two_write_queues_ 或 unordered_write 启用时使用
IOStatus DBImpl::ConcurrentWriteToWAL(
    const WriteThread::WriteGroup& write_group, uint64_t* log_used,
    SequenceNumber* last_sequence, size_t seq_inc) {
  IOStatus io_s;

  assert(two_write_queues_ || immutable_db_options_.unordered_write);
  assert(!write_group.leader->disable_wal);
  // 合并事务组中的多个 WriteBatch，提高 WAL 写入效率
  WriteBatch tmp_batch;
  size_t write_with_wal = 0;
  WriteBatch* to_be_cached_state = nullptr;
  WriteBatch* merged_batch;
  io_s = status_to_io_status(MergeBatch(write_group, &tmp_batch, &merged_batch,
                                        &write_with_wal, &to_be_cached_state));
  if (UNLIKELY(!io_s.ok())) {
    return io_s;
  }

  // 由于是多线程需要获取锁
  log_write_mutex_.Lock();
  
  if (merged_batch == write_group.leader->batch) {
    // Leader 线程的 log_used 记录当前 logfile_number_
    write_group.leader->log_used = logfile_number_;
  } else if (write_with_wal > 1) {
    // 如果多个事务写入 WAL，所有 write_group 的事务共享相同的 logfile_number_
    for (auto writer : write_group) {
      writer->log_used = logfile_number_;
    }
  }
  // 设置 WAL 全局事务序列号
  *last_sequence = versions_->FetchAddLastAllocatedSequence(seq_inc);
  auto sequence = *last_sequence + 1;
  WriteBatchInternal::SetSequence(merged_batch, sequence);

  // 从 logs_（日志文件列表） 获取最新的 log_writer
  log::Writer* log_writer = logs_.back().writer;
  LogFileNumberSize& log_file_number_size = alive_log_files_.back();

  // 真正执行 WAL 写入
  uint64_t log_size;
  WriteOptions write_options;
  write_options.rate_limiter_priority =
      write_group.leader->rate_limiter_priority;
  io_s = WriteToWAL(*merged_batch, write_options, log_writer, log_used,
                    &log_size, log_file_number_size);
  log_write_mutex_.Unlock();
  // ...
}


IOStatus DBImpl::WriteToWAL(const WriteBatch& merged_batch,
                            const WriteOptions& write_options,
                            log::Writer* log_writer, uint64_t* log_used,
                            uint64_t* log_size,
                            LogFileNumberSize& log_file_number_size) {
  // 获取 WriteBatch 日志内容
  Slice log_entry = WriteBatchInternal::Contents(&merged_batch);

  *log_size = log_entry.size();
  const bool needs_locking = manual_wal_flush_ && !two_write_queues_;
  if (UNLIKELY(needs_locking)) {
    log_write_mutex_.Lock();
  }
  // 添加时间戳
  IOStatus io_s = log_writer->MaybeAddUserDefinedTimestampSizeRecord(
      write_options, versions_->GetColumnFamiliesTimestampSizeForRecord());
  if (!io_s.ok()) {
    return io_s;
  }
  // 写入 WAL
  io_s = log_writer->AddRecord(write_options, log_entry);

  if (UNLIKELY(needs_locking)) {
    log_write_mutex_.Unlock();
  }
  // 记录文件变化
  if (log_used != nullptr) {
    *log_used = logfile_number_;
  }
  // ...
}
```



#### 删除 WAL

在 [MANIFEST:ProcessManifestWrites](https://github.com/LiuRuoyu01/learn-rocksdb/blob/main/ch02/RocksDB_Manifest.md#ProcessManifestWrites) 章节中，更新 MANIFEST 的同时会[更新 min_log_number](https://github.com/LiuRuoyu01/learn-rocksdb/blob/main/ch02/RocksDB_Manifest.md#manifest-%E8%B7%9F%E8%B8%AAwal-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F)，`FindObsoleteFiles` 会根据此信息取判断 WAL 生命周期是否结束，从而删除 WAL 文件

```c++
void DBImpl::FindObsoleteFiles(JobContext* job_context, bool force,
                               bool no_full_scan) {
  // ...
  job_context->log_number = versions_->min_log_number_to_keep();
  // ...
  log_write_mutex_.Lock();

  if (alive_log_files_.empty() || logs_.empty()) {
    log_write_mutex_.Unlock();
    return;
  }

  if (!alive_log_files_.empty() && !logs_.empty()) {
    // logfile_num <= min_log_number 都应该删除
    uint64_t min_log_number = job_context->log_number;
    size_t num_alive_log_files = alive_log_files_.size();

    // 1. 检测新的 obsoleted WAL files，即生命周期已结束的
    while (alive_log_files_.begin()->number < min_log_number) {
      auto& earliest = *alive_log_files_.begin();

      // 是否回收，不收回则加入待删除队列
      if (immutable_db_options_.recycle_log_file_num > log_recycle_files_.size()) {
        log_recycle_files_.push_back(earliest.number);
      } else {
        job_context->log_delete_files.push_back(earliest.number);
      }
      alive_log_files_.pop_front();
    }
    log_write_mutex_.Unlock();
    mutex_.Unlock();

    // 2. 检测待释的 LogWriter
    log_write_mutex_.Lock();
    while (!logs_.empty() && logs_.front().number < min_log_number) {
      auto& log = logs_.front();
      if (log.IsSyncing()) {
        log_sync_cv_.Wait();
        continue;
      }
      logs_to_free_.push_back(log.ReleaseWriter());
      logs_.pop_front();
    }
  }

  job_context->logs_to_free = logs_to_free_;
  logs_to_free_.clear();
  log_write_mutex_.Unlock();

  // 3. 回收
  mutex_.Lock();
  job_context->log_recycle_files.assign(log_recycle_files_.begin(),
                                        log_recycle_files_.end());
}
```





