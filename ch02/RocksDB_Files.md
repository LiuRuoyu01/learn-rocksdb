## 文件介绍

```
//插入一些数据
(base) ➜  tools git:(v9.9.3) ./ldb --db=my_rocksdb --create_if_missing put key1 value1
...

(base) ➜  tools git:(v9.9.3) cd my_rocksdb
(base) ➜  my_rocksdb git:(v9.9.3) ll
total 228K
//sst文件
-rw-r--r-- 1 liu liu 1.1K Dec 30 15:33 000008.sst          	
-rw-r--r-- 1 liu liu 1.1K Dec 30 15:33 000013.sst
-rw-r--r-- 1 liu liu 1.1K Dec 30 15:33 000018.sst
-rw-r--r-- 1 liu liu 1.1K Dec 30 15:34 000023.sst
-rw-r--r-- 1 liu liu 1.1K Dec 30 15:35 000030.sst
//WAL日志，记录未刷盘到 SST 文件的写入操作
-rw-r--r-- 1 liu liu   30 Dec 30 15:35 000031.log	
//当前使用的 MANIFEST 文件（例如 MANIFEST-000032 ）
-rw-r--r-- 1 liu liu   16 Dec 30 15:35 CURRENT							
-rw-r--r-- 1 liu liu   36 Dec 30 15:33 IDENTITY
-rw-r--r-- 1 liu liu    0 Dec 30 15:32 LOCK
-rw-r--r-- 1 liu liu  27K Dec 30 15:35 LOG
//多个 LOG.old.* 文件表明 RocksDB 启动或发生了一些活动（如 Compaction）
-rw-r--r-- 1 liu liu 9.1K Dec 30 15:32 LOG.old.1735543983570149
-rw-r--r-- 1 liu liu  23K Dec 30 15:33 LOG.old.1735544024061407
-rw-r--r-- 1 liu liu  26K Dec 30 15:33 LOG.old.1735544029820178
-rw-r--r-- 1 liu liu  26K Dec 30 15:33 LOG.old.1735544035957524
-rw-r--r-- 1 liu liu  26K Dec 30 15:33 LOG.old.1735544040605367
-rw-r--r-- 1 liu liu  26K Dec 30 15:34 LOG.old.1735544128835729
//存储了数据库的元数据，包括 SST 文件的层级、键值范围和存储位置
-rw-r--r-- 1 liu liu  920 Dec 30 15:35 MANIFEST-000032			
//OPTIONS 文件 记录了 RocksDB 的配置信息
-rw-r--r-- 1 liu liu 7.5K Dec 30 15:34 OPTIONS-000027
-rw-r--r-- 1 liu liu 7.5K Dec 30 15:35 OPTIONS-000034

```



## WAl文件

记录__写操作日志__的文件。它的主要功能是确保在数据库发生故障（如宕机或崩溃）时，可以通过日志恢复未刷入 SST 文件的数据，从而避免数据丢失

``` 
./ldb dump_wal --walfile=my_rocksdb/000031.log --header --print_value
打印如下 
Sequence,Count,ByteSize,Offset,Physical  Key(s) : value     
1,       1,    25,      0,      PUT(0) : 0x6B657932 : 0x76616C756532 
2,       1,    25,      32,     PUT(0) : 0x6B657933 : 0x76616C756534 
以第一行为例：序列号:1，当前操作的个数:1，字节数:25，物理偏移:0，PUT(0)表示写入操作，写入列族id为0，即默认列族，后面分别是key的值和value的值，0x6B657932转为ascii即key2，0x6B657932为value2
```



## MemTable文件

RocksDB的写请求写入到MemTable后就认为是写成功了，MemTable存放在内存中的，他保存了__落盘到SST文件前的数据__。同时服务于读和写，新的写入总是将数据插入到MemTable。一旦一个MemTable被写满（或者满足一定条件），他会变成不可修改的MemTable，即ImMemTable，并被一个新的MemTable替换。一个后台线程会将ImMemTable的内容落盘到一个SST文件，然后ImMemTable就可以被销毁了。

单个memtable的key分布是有序的，最常用的是基于SkipList（跳表）实现，有较好的读写性能，并支持并发写入

<img src="./images/memtable.png" alt="memtable" style="zoom:200%;" />

```c++
//存入MemTable的kv数据格式
|-internal_key_size-|---key---|--seq--type|--value_size--|--value--|
//internal_key_size : varint类型，包括key、seq、type所占的字节数
//key：字符串，就是Put进来的key字符串seq：
//序列号，占7个字节type：
//操作类型，占1个字节（Put or Dlete）
//value_size：varint类型，表示value的长度
//value：字符串，就是Put进来的value字符串
```

```c++
class MemTable{  
  ...  
  KeyComparator comparator_; //用于比较key的大小  	
  std::unique_ptr<MemTableRep> table_; //指向skiplist  
  std::unique_ptr<MemTableRep> range_del_table_;//指向skiplist，用于kTypeRangeDeletion类型(memtable支持范围删除)

  // Total data size of all data inserted  
  std::atomic<uint64_t> data_size_;  
  std::atomic<uint64_t> num_entries_;  
  std::atomic<uint64_t> num_deletes_;    
  
  // Dynamically changeable memtable option  
  std::atomic<size_t> write_buffer_size_;	//可支持最大写数据的大小
  
  bool flush_in_progress_; // started the flush  
  bool flush_completed_;   // finished the flush  
  uint64_t file_number_;    // filled up after flush is complete
  
  // The updates to be applied to the transaction log when this  
  // memtable is flushed to storage.  
  VersionEdit edit_;  			//版本
  std::unique_ptr<DynamicBloom> bloom_filter_; //布隆过滤器（可快速判断kv是否存在）
}

bool MemTable::Add(SequenceNumber s, ValueType type, const Slice& key, /* user key */  
                   const Slice& value, bool allow_concurrent,
                   MemTablePostProcessInfo*post_process_info, void** hint) {  
  //kv编码     
  // Format of an entry is concatenation of:   
  //  key_size     : varint32 of internal_key.size()    
  //  key bytes    : char[internal_key.size()]  
  //  value_size   : varint32 of value.size()  
  //  value bytes  : char[value.size()]  
  uint32_t key_size = static_cast<uint32_t>(key.size());  
  uint32_t val_size = static_cast<uint32_t>(value.size());  
  uint32_t internal_key_size = key_size + 8;  		//8是 序列号的7字节 + 操作形式的1字节
  const uint32_t encoded_len = VarintLength(internal_key_size) + 
    internal_key_size + VarintLength(val_size) +                               
    val_size; 
  char* buf = nullptr;  
  
  // 通过判断key-value的类型来选择memtable, 范围删除的kv插入range_del_table_
  std::unique_ptr<MemTableRep>& table =      
    type == kTypeRangeDeletion ? range_del_table_ : table_;
  
  //申请内存空间，并将数据拷贝到内存中去
  KeyHandle handle = table->Allocate(encoded_len, &buf);
  char* p = EncodeVarint32(buf, internal_key_size);  
  memcpy(p, key.data(), key_size);  
  Slice key_slice(p, key_size);  
  p += key_size;  
  uint64_t packed = PackSequenceAndType(s, type);  
  EncodeFixed64(p, packed);  
  p += 8;  
  p = EncodeVarint32(p, val_size);  
  memcpy(p, value.data(), val_size);  
  assert((unsigned)(p + val_size - buf) == (unsigned)encoded_len);
  size_t ts_sz = GetInternalKeyComparator().user_comparator()->timestamp_size();  
  
  // allow_concurrent默认为false 
  //是否开启并发写入
  if (!allow_concurrent) {    
  	// Extract prefix for insert with hint.  
    //带hint插入，通过map记录一些前缀插入skiplist的位置，从而再次插入相同前缀的key时快速找到位置    
    //默认不启用   
    if (insert_with_hint_prefix_extractor_ != nullptr && 
        insert_with_hint_prefix_extractor_->InDomain(key_slice)) {      
      Slice prefix = insert_with_hint_prefix_extractor_->Transform(key_slice);  
      bool res = table->InsertKeyWithHint(handle,&insert_hints_[prefix]);      
      if (UNLIKELY(!res)) {        
          return res;      
      }    
    } else {   
      //插入到skiplist   
      bool res = table->InsertKey(handle); 
      if (UNLIKELY(!res)) {        
        return res;      
      }   
    }
    
    // 更新统计信息
    num_entries_.store(num_entries_.load(std::memory_order_relaxed) + 1,    
                       std::memory_order_relaxed);
    data_size_.store(data_size_.load(std::memory_order_relaxed) + encoded_len,
                       std::memory_order_relaxed);   
    if (type == kTypeDeletion) {      
        num_deletes_.store(num_deletes_.load(std::memory_order_relaxed) + 1,             
                           std::memory_order_relaxed);    
    }    
    
    //更新布隆过滤器    
    if (bloom_filter_ && prefix_extractor_ &&        
          prefix_extractor_->InDomain(key)) {      
        bloom_filter_->Add(prefix_extractor_->Transform(key));
    }
    
    if (bloom_filter_ && 
      moptions_.memtable_whole_key_filtering) {      
        bloom_filter_->Add(StripTimestampFromUserKey(key, ts_sz));    
    }
    
    // The first sequence number inserted into the memtable   
    //确保内存表中序号的一致性
    //第一个序号为0或者当前操作序号大于等于第一个序号
    assert(first_seqno_ == 0 || s >= first_seqno_);    
    if (first_seqno_ == 0) {      
      first_seqno_.store(s, std::memory_order_relaxed);
      if (earliest_seqno_ == kMaxSequenceNumber) {        
        earliest_seqno_.store(GetFirstSequenceNumber(),                              
                              std::memory_order_relaxed);      
      }      
        assert(first_seqno_.load() >= earliest_seqno_.load());    
    }    
    assert(post_process_info == nullptr);    
    //更新内存表的刷新状态
    UpdateFlushState(); 
  } else {    
      // 并发插入    
      ...  
  }  
  return true;
}

bool MemTable::Get(const LookupKey& key, std::string* value,
                   PinnableWideColumns* columns, std::string* timestamp,
                   Status* s, MergeContext* merge_context,
                   SequenceNumber* max_covering_tombstone_seq,
                   SequenceNumber* seq, const ReadOptions& read_opts,
                   bool immutable_memtable, ReadCallback* callback,
                   bool* is_blob_index, bool do_merge) {
  
  // ...
	
  // 在range_del_table_上初始化一个迭代器，用于遍历范围删除的记录
  std::unique_ptr<FragmentedRangeTombstoneIterator> range_del_iter(
      NewRangeTombstoneIterator(read_opts,
                                GetInternalKeySeqno(key.internal_key()),
                                immutable_memtable));
  if (range_del_iter != nullptr) {
    //获取范围删除中包含此键的最大序号
    SequenceNumber covering_seq =
        range_del_iter->MaxCoveringTombstoneSeqnum(key.user_key());
    //如果删除的序号大于此序号，则范围删除优先级最高
    if (covering_seq > *max_covering_tombstone_seq) {
      *max_covering_tombstone_seq = covering_seq;
      //更新时间戳
    }
  }

  //...
  
  //用布隆过滤器判断键是否可能存在memtable里面
  if (bloom_filter_) {
    // 全键过滤
    if (moptions_.memtable_whole_key_filtering) {
      may_contain = bloom_filter_->MayContain(user_key_without_ts);
      bloom_checked = true;
    } else {
      //如果设置了前缀提词器则对前缀进行过滤，前缀过滤器通常用于范围查询
      assert(prefix_extractor_);
      if (prefix_extractor_->InDomain(user_key_without_ts)) {
        may_contain = bloom_filter_->MayContain(
            prefix_extractor_->Transform(user_key_without_ts));
        bloom_checked = true;
      }
    }
  }
	
  if (bloom_filter_ && !may_contain) {
    // 如果布隆过滤器判断键不在，则键肯定不存在
    PERF_COUNTER_ADD(bloom_memtable_miss_count, 1);
    *seq = kMaxSequenceNumber;
  } else {
    if (bloom_checked) {
      PERF_COUNTER_ADD(bloom_memtable_hit_count, 1);
    }
    //进行精确查找
    GetFromTable(key, *max_covering_tombstone_seq, do_merge, callback,
                 is_blob_index, value, columns, timestamp, s, merge_context,
                 seq, &found_final_value, &merge_in_progress);
  }

  //...
  
  PERF_COUNTER_ADD(get_from_memtable_count, 1);
  return found_final_value;
}
```

```c++
char* InlineSkipList<Comparator>::AllocateKey(size_t key_size) {  
  //这里会随机一个高度，也就是跳表里面一个节点的高度  
  return const_cast<char*>(AllocateNode(key_size, RandomHeight())->Key());
}

//为一个新的跳表节点分配内存
InlineSkipList<Comparator>::AllocateNode(size_t key_size, int height) {  
  //每个指针指向该高度的下一个节点高度，最底下的节点暂时无指向
  auto prefix = sizeof(std::atomic<Node*>) * (height - 1);
  //通过Arena::AllocateAligned或者ConcurrentArena::AllocateAligned去申请内存  
  char* raw = allocator_->AllocateAligned(prefix + sizeof(Node) + key_size);  

  //将节点高度暂时存储在高度为1的位置，插入跳表完成后就不需要高度了,这个位置就会存放指向下一个节点的指针 
  Node* x = reinterpret_cast<Node*>(raw + prefix);
  x->StashHeight(height); 
  
  return x;
}
```



## manifest文件

manifest是元数据文件，描述了所有列族中__LSM树结构信息__的文件，每一层中的sst文件数量，以及每一层sst文件概要信息，可以通过MANIFEST观察当前LSM树的结构，宕机后通过manifest文件还原LSM树

``` c++
./ldb manifest_dump --path="my_rocksdb/MANIFEST-000032"
//打印如下：
--------------- Column family "default"  (ID 0) --------------   
log number: 10    				//正在使用WAL文件的编号
comparator: leveldb.BytewiseComparator     //RocksDB默认比较器，按照字符大小比较键值
--- level 0 --- version# 1 ---  
//SST文件编号，SST文件大小，SST文件的序号，  SST文件的最小key和最大key
  10:        13759       [2049 .. 3072]['key0' seq:2049, type:0 .. 'key999' seq:3048, type:0]  
  7:         1076837     [1025 .. 2048]['key0' seq:1025, type:1 .. 'key999' seq:2024, type:1]     
  4:         1076837     [1 .. 1024]   ['key0' seq:1, type:1 .. 'key999' seq:1000, type:1]    
--- level 1 --- version# 1 ---    
--- level 2 --- version# 1 ---    
.....//省略    
--- level 62 --- version# 1 ---    
--- level 63 --- version# 1 ---
next_file_number 46 				//表示下一个sst文件可用的编号为46
last_sequence 3072  				//表示上次的写操作的序列号为3072
prev_log_number 0 					//表示当前WAL文件之前的一个WAL文件的编号，确保日志文件的顺序和一致性
max_column_family 0 				//最大的列族编号，这里是0（只有一个默认列族）
min_log_number_to_keep 0		//2PC模式下使用，恢复过程中忽略小于等于该值的日志

```


