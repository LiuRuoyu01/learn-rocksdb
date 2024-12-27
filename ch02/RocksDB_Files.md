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



## Bloom过滤器

#### 作用

1. 判断一个元素肯定不在一个集合中
2. 判断一个元素可能在一个集合中

#### 原理

布隆过滤器由**一个长度为m的位数组和k个哈希函数**组成。

- 在添加key时，先通过k个hash函数计算出在位数组上的k个位置，然后将数组对应的位置记为1。

- 在查找key时，先通过k个hash函数计算出在位数组上的k个位置，判断数组对应的每个位置的值是否为1，如果全部为1说明元素**可能**在集合中，反之**一定**不在集合中

**RocksDB通过减少布隆过滤器的误判率进行性能调优**

```c++
//创建布隆过滤器
//bits_per_key即一个key占用多少个bit
//use_block_based_builder是否使用旧版本基于块的Bloom过滤器实现模式
const FilterPolicy* NewBloomFilterPolicy(double bits_per_key,                                         bool use_block_based_builder) {  
  BloomFilterPolicy::Mode m;  
  //一般都用kAuto模式  
  if (use_block_based_builder) {    
    m = BloomFilterPolicy::kDeprecatedBlock;  
  } else {    
    m = BloomFilterPolicy::kAuto; 
  }  
  assert(std::find(BloomFilterPolicy::kAllUserModes.begin(),                   
                   BloomFilterPolicy::kAllUserModes.end(),                  
                   m) != BloomFilterPolicy::kAllUserModes.end());  
  return new BloomFilterPolicy(bits_per_key, m);
}

//判断key是否有可能包含在集合内
bool BloomFilterPolicy::KeyMayMatch(
  const Slice& key,                                    
  const Slice& bloom_filter) const {  
  //部分代码省略  
  const size_t len = bloom_filter.size();  
  const char* array = bloom_filter.data();  
  const uint32_t bits = static_cast<uint32_t>(len - 1) * 8;  
  const int k = static_cast<uint8_t>(array[len - 1]);   
  return LegacyNoLocalityBloomImpl::HashMayMatch(BloomHash(key), bits, k,                                                 array);
}
// h:            根据key计算出的hash值  
// total_bits:   bloom过滤器位数组长度  
// num_probes：嗅探次数，即一个key最终映射到布隆过滤器位数组上的bit个数，也就是k  
// data：布隆过滤器位数组  
static inline bool HashMayMatch(uint32_t h, uint32_t total_bits,                   
                                int num_probes, const char *data) {  
  
  //循环位移，用delta模拟多个独立的hash函数
  const uint32_t delta = (h >> 17) | (h << 15);   
  
  for (int i = 0; i < num_probes; i++) {      
    const uint32_t bitpos = h % total_bits;      
    //这里以一个字节去比较对应位，只要有一个bit不为1就直接返回false      
    // (1 << (bitpos % 8))  该字节上对应pos置为1      
    // data[bitpos / 8]  布隆过滤器位数组上该bit所属的对应字节      
    // if也就是判断布隆过滤器位数组上对应pos的位是否为0，      
    // 是说明该key不在集合内，直接返回false      
    if ((data[bitpos / 8] & (1 << (bitpos % 8))) == 0) {        
      return false;      
    }      
    h += delta;    
  }    
  return true;  
};

//添加一个key到布隆过滤器内
static inline void AddHash(uint32_t h, uint32_t total_bits, int num_probes,                             char *data) {    
  const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits    
  for (int i = 0; i < num_probes; i++) {      
    const uint32_t bitpos = h % total_bits;      
    data[bitpos / 8] |= (1 << (bitpos % 8));      
    h += delta;    
  }
}
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



## SST文件

在rocksdb内通过flush和compaction生成SST文件， SST 文件是用来存储一系列**有序的 KV对**，Key 和 Value 都是任意长度的字符串；在rocksdb层级里面，L0的各个SST文件之间是**没有严格排序**的，而L1及L1+层级中SST文件之间是**严格有序**的

<img src="./images/SST.png" alt="SST" style="zoom:150%;" />

```c++
sst_dump解析sst文件
./sst_dump --file=my_rocksdb/000008.sst --command=raw
处理成功会有如下打印，并写入到000008_dump.txt文件
from [] to []
Process my_rocksdb/sst_file.sst
Sst file format: block-based
raw dump written to file my_rocksdb/000008_dump.txt

输出解析的sst文件内容
cat 000008_dump.txt
 
Footer Details:
--------------------------------------  
checksum: 1                 			//校验和  
metaindex handle: D10736   				//索引Metaindex Details  
index handle: 9F3A16         				//索引Index 
Details  footer version: 6        //版本  
table_magic_number: 9863518390377041911  //固定值，验证文件是否为合法的 SST 文件  
  
//元数据
Metaindex Details:
--------------------------------------  
Filter block handle: C30345            // 索引布隆过滤器数据块  
Properties block handle: F204C805      // 索引Table Properties  属性
Range deletion block handle: AF043E    // 索引Range deletions		范围删除

Table Properties:
--------------------------------------  
# data blocks: 1                                     // Data Block个数  
# entries: 22                                        // 条目数 20put + 2delete_range  
# deletions: 2                                       // deletion个数 deletion + deletion range  
# merge operands: 0                                  // merge操作个数  
# range deletions: 2                                 // deletion range个数  
raw key size: 330                                    // 原始key大小  
raw average key size: 15.000000                      // 平均每个key占用的空间大小  
raw value size: 194                                  // 原始value大小  
raw average value size: 8.818182                     // 平均每个value占用的空间大小  
data block size: 451                                 // data block大小  
index block size (user-key? 0, delta-value? 0): 34   // index block大小  
filter block size: 69                                // filter block大小  
(estimated) table size: 554                          // 预估table大小  
filter policy name: rocksdb.BuiltinBloomFilter       // 过滤器名称  
prefix extractor name: nullptr                       // 前缀提取器名称  
column family ID: N/A                                // 列族ID，这里直接写的sst，没有列族  
column family name: N/A                              // 列族名称  
comparator name: leveldb.BytewiseComparator          // 比较器名称  
merge operator name: nullptr                         // 合并操作符名称  
property collectors names: []                        // 属性收集器名称  
SST file compression algo: NoCompression             // 压缩方式，不压缩  
SST file compression options: window_bits=-14; level=32767; strategy=0; max_dict_bytes=0; zstd_max_train_bytes=0; enabled=0;   
creation time: 0                                     // 最先写入memtable的时间  
time stamp of earliest key: 0                        //   
file creation time: 0                                // sst文件创建时间  

//索引块
Index Details:    
--------------------------------------  
Block key hex dump: Data block handle  
Block key ascii
HEX    6B65795F626239: 00BE03  
ASCII  k e y _ b b 9   
------

//范围删除数据
Range deletions: 
--------------------------------------    
HEX    6B65795F626232: 6B65795F626235  
ASCII  k e y _ b b 2 : k e y _ b b 5   		//bb2-bb5全部是删除
------ 
...

//key value
Data Block # 1 @ 00BE03  
--------------------------------------  
HEX    6B65795F616130: 76616C75655F616130  
ASCII  k e y _ a a 0 : v a l u e _ a a 0   
------  
HEX    6B65795F616131: 76616C75655F616131  
ASCII  k e y _ a a 1 : v a l u e _ a a 1   
------
...

Data Block Summary:
--------------------------------------  
# data blocks: 1  
min data block size: 446         // 最小block大小  
max data block size: 446         // 最大block大小  
avg data block size: 446.000000  // 平均block大小
```

