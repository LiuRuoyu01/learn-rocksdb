## Bloom Filter

#### 作用

1. 判断一个元素**肯定不在**一个集合中
2. 判断一个元素**可能在**一个集合中

#### 原理

布隆过滤器由**一个长度为m的位数组和k个哈希函数**组成。

- 在添加key时，先通过k个hash函数计算出在位数组上的k个位置，然后将数组对应的位置记为1。

- 在查找key时，先通过k个hash函数计算出在位数组上的k个位置，判断数组对应的每个位置的值是否为1，如果全部为1说明元素**可能**在集合中，反之**一定**不在集合中

在 RocksDB 中，当设置了过滤策略时，每个新创建的 SST 文件都会包含一个布隆过滤器，该过滤器嵌入在 SST 文件本身中，用于确定文件是否可能包含我们要查找的键。过滤器本质上是一个位数组。多个哈希函数应用于给定的键，每个函数指定数组中将设置为 1 的位。在读取时，也会对搜索键应用相同的哈希函数，检查位，即探测，如果至少一个探测返回 0，则该键肯定不存在。**RocksDB通过减少布隆过滤器的误判率进行性能调优**



#### Block-based Bloom Filter（old format）

为每个 **数据块** 构建布隆过滤器，它按照 SST 文件的 **字节偏移范围** 将键值分组，并为每个范围生成一个独立的过滤器。

```
+----------------------------------------+
| filter 0 data (256 bytes)              |        //布隆过滤器的数据部分，256bytes表示这部分大小
+----------------------------------------+
| filter 1 data (256 bytes)              |
+----------------------------------------+
| filter 2 data (256 bytes)              |
+----------------------------------------+
| padding (256 bytes)                    |        //内存对齐，
+----------------------------------------+
| offset of filter 0 = 0        (4 byte) |        // 偏移量数组，过滤器在此block的位置
+----------------------------------------+
| offset of filter 1 = 256      (4 byte) |						
+----------------------------------------+
| offset of filter 2 = 512      (4 byte) |
+----------------------------------------+
| offset of offset array = 1024 (4 byte) |        //偏移量数组起始位置
+----------------------------------------+
| lg(base) = 11                 (1 byte) |        // base = 2^11 每个过滤器覆盖2KB的偏移范围
+----------------------------------------+
```

```
Legacy Bloom filter data:
              0 +-----------------------------------+
                | Raw Bloom filter data             |     //存储布隆过滤器的位数组
                | Raw Bloom filter data             |	
                | ...                               |
            len +-----------------------------------+
                | byte for num_probes or            |     //哈希探测次数
                |   marker for new implementations  |     //标识布隆过滤器是否使用新实现
          len+1 +-----------------------------------+
                | four bytes for number of cache    |     //缓存行数量
                |   lines                           |     //对齐缓存行，以优化内存访问
  len_with_meta +-----------------------------------+


```

存在的问题：

1. 由于需要找到key属于对应的哪个块再找的对应的fliter，所以无论布隆过滤器是否过滤掉此key都需要加载数据的index
2. 由于filter block可能过大，访问其中的过滤器可能导致内存随机访问，各个过滤器块未与缓存对齐，因此在查找过程中可能会导致大量缓存未命中
3. 范围查询效率低，需要跨多个数据块

```c++
// LegacyBloomBitsBuilder
// 添加数据
void LegacyBloomBitsBuilder::AddKey(const Slice& key) {
  uint32_t hash = BloomHash(key);
  if (hash_entries_.empty() || hash_entries_.back() != hash) {
    hash_entries_.push_back(hash);
  }
}

// 返回一个 Slice 对象，包含布隆过滤器的位数组及元数据,位数组保存在buf里面
Slice LegacyBloomBitsBuilder::Finish(std::unique_ptr<const char[]>* buf) {
  size_t num_entries = hash_entries_.size();
	
  // 分配位数组空间
  uint32_t total_bits, num_lines;
  char* data =
      ReserveSpace(static_cast<int>(num_entries), &total_bits, &num_lines);
  assert(data);

  if (total_bits != 0 && num_lines != 0) {
    for (auto h : hash_entries_) {
      // 添加哈希值
      AddHash(h, data, num_lines, total_bits);
    }
    // 检查误判率 
  }
  
  // 添加元数据
  // num_probes_ ： 哈希探测次数
  // num_lines   ： 缓存行数量
  data[total_bits / 8] = static_cast<char>(num_probes_);
  EncodeFixed32(data + total_bits / 8 + 1, static_cast<uint32_t>(num_lines));

  const char* const_data = data;
  buf->reset(const_data);
  hash_entries_.clear();
  prev_alt_hash_ = -1;
  
	// 数据的起始地址data，数据的长度（total_bits / 8 + kMetadataLen）
  return Slice(data, total_bits / 8 + kMetadataLen);
}

// 添加一个key到布隆过滤器内
static inline void AddHash(uint32_t h, uint32_t num_lines, int num_probes,
                             char *data, int log2_cache_line_bytes) {
  // 数组通常存在为字节数组中，由于操作目标是位数组（bite），需要避免跨字节的操作，
  // 3表示log2(8)，需要1字节空间避免溢出或越界（比如511bit这种情况）
  const int log2_cache_line_bits = log2_cache_line_bytes + 3;
  
  // 定位数据的起始位置
  char *data_at_offset =
        data + (GetLine(h, num_lines) << log2_cache_line_bytes);
  
  // 循环位移，用delta模拟多个独立的hash函数
  const uint32_t delta = (h >> 17) | (h << 15);
  for (int i = 0; i < num_probes; ++i) {
    const uint32_t bitpos = h & ((1 << log2_cache_line_bits) - 1);
    data_at_offset[bitpos / 8] |= (1 << (bitpos % 8));
    if (ExtraRotates) {
      h = (h >> log2_cache_line_bits) | (h << (32 - log2_cache_line_bits));
    }
    h += delta; 
  }
}

```

```c++
// LegacyBloomBitsBuilder
// 查询数据
// 判断key是否有可能包含在集合内
bool MayMatch(const Slice& key) override {
  uint32_t hash = BloomHash(key);
  uint32_t byte_offset;
  //预取，减少查询时的缓存未命中开销
  //由于位数组通常存储在内存中，频繁的随机访问可能导致大量缓存未命中
  //提前将相关内存加载到 CPU 缓存中，降低访问延迟
  LegacyBloomImpl::PrepareHashMayMatch(
        hash, num_lines_, data_, /*out*/ &byte_offset, log2_cache_line_size_);
  return LegacyBloomImpl::HashMayMatchPrepared(
        hash, num_probes_, data_ + byte_offset, log2_cache_line_size_);
}


// h:                     根据key计算出的hash值  
// num_probes:            嗅探次数，即一个key最终映射到布隆过滤器位数组上的bit个数 
// data_at_offset:        布隆过滤器位数组当前缓存行起始位置
// log2_cache_line_bytes: 缓存行大小的对数值
static inline bool HashMayMatchPrepared(uint32_t h, int num_probes,
                                          const char *data_at_offset,
                                          int log2_cache_line_bytes) {
  // 数组通常存在为字节数组中，由于操作目标是位数组（bite），需要避免跨字节的操作，
  // 3表示log2(8)，需要1字节空间避免溢出或越界（比如511bit这种情况）
  const int log2_cache_line_bits = log2_cache_line_bytes + 3;
	
  // 循环位移，用delta模拟多个独立的hash函数
  const uint32_t delta = (h >> 17) | (h << 15);
  
  for (int i = 0; i < num_probes; ++i) {
    const uint32_t bitpos = h & ((1 << log2_cache_line_bits) - 1); 
    // 这里以一个字节去比较对应位，只要有一个bit不为1就直接返回false      
    // (1 << (bitpos % 8))  该字节上对应pos置为1      
    // data[bitpos / 8]  布隆过滤器位数组上该bit所属的对应字节      
    // if也就是判断布隆过滤器位数组上对应pos的位是否为0，      
    // 是说明该key不在集合内，直接返回false      
    if (((data_at_offset[bitpos / 8]) & (1 << (bitpos % 8))) == 0) {    
      return false；      
    }   
    // 是否更新哈希值，增加哈希值的分布随机性
    if (ExtraRotates) {  
      h = (h >> log2_cache_line_bits) | (h << (32 - log2_cache_line_bits));      
    }     
    h += delta;
  }    
  return true;
}

```



#### Full Filters (new format)

- 通过为**整个SST文件**创建一个过滤器来解决block filter所产生的问题1，代价是需要更多内存来缓存布隆过滤器的。
- 为了解决问题2，Full Filters利用CPU缓存局部性来提高效率。限制每个键的哈希探测位位于**同一个缓存行**，避免跨行访问

如下图所示，位置(0 - len)为过滤器位数组n个数据块的位置，位置(len2 - len3)规定了数据块的大小，通常块大小为CPU缓存行大小。

查询时，首先通过计算key的hash值确定此key映射到其中一个filter位数组块中，Full Filter 将每个键的所有哈希探测限制在布隆过滤器位数组的同一个块中，避免了跨缓存行的访问，减少了CPU缓存未命中的概率，之后每次hash都在此块进行查找

```
New Bloom filter data:
            0 +-----------------------------------+     //存储布隆过滤器的位数组
              | Raw Bloom filter data  1          |
              | Raw Bloom filter data  2          |
              | ...                               |
          len +-----------------------------------+     //标志字节，例如-1表示新格式	
              | char{-1} byte -> new Bloom filter |     //区分新旧实现
        len+1 +-----------------------------------+
              | byte for subimplementation        |     //标志字节，支持多种过滤器实现
              |   0: FastLocalBloom               |     //例如0表示 FastLocalBloom 实现
              |   other: reserved                 |
        len+2 +-----------------------------------+
              | byte for block_and_probes         |     //存储块大小和哈希探测次数
              |   0 in top 3 bits -> 6 -> 64-byte |     //高三位表示的是布隆过滤器块大小
              |   reserved:                       |     //块大小为2的（6+vlaue）次方
              |   1 in top 3 bits -> 7 -> 128-byte|     //块最大大小为8KB
              |   2 in top 3 bits -> 8 -> 256-byte|
              |   ...                             |
              |   num_probes in bottom 5 bits,    |     //低五位是哈希探测次数（1-30）
              |     except 0 and 31 reserved      |
        len+3 +-----------------------------------+
              | two bytes reserved                |     //预留两字节方便未来扩展
              |   possibly for hash seed          |
len_with_meta +-----------------------------------+
```

```c++
// FastLocalBloomBitsBuilder
// 添加数据
void FullFilterBlockBuilder::Add(const Slice& key_without_ts) {
  //是否适合前缀提取
  if (prefix_extractor_ && prefix_extractor_->InDomain(key_without_ts)) {
    Slice prefix = prefix_extractor_->Transform(key_without_ts);
    //是否添加完整键
    if (whole_key_filtering_) {
      filter_bits_builder_->AddKeyAndAlt(key_without_ts, prefix);
    } else {
      filter_bits_builder_->AddKey(prefix);
    }
  } else if (whole_key_filtering_) {
    filter_bits_builder_->AddKey(key_without_ts);
  }
}

void AddHash(uint64_t hash) {
  // 过滤器校验
  if (detect_filter_construct_corruption_) {
    hash_entries_info_.xor_checksum ^= hash;
  }
  // 存储哈希值
  hash_entries_info_.entries.push_back(hash);
  
  // 每当当前存储的哈希值数量达到缓存桶的一半时，就申请一个新的缓存桶
  if (cache_res_mgr_ &&
      ((hash_entries_info_.entries.size() %
        kUint64tHashEntryCacheResBucketSize) ==
       kUint64tHashEntryCacheResBucketSize / 2)) {
    hash_entries_info_.cache_res_bucket_handles.emplace_back(nullptr);
    Status s = cache_res_mgr_->MakeCacheReservation(
      kUint64tHashEntryCacheResBucketSize * sizeof(hash),         
      &hash_entries_info_.cache_res_bucket_handles.back());
      s.PermitUncheckedError();
  }
}

Slice Finish(std::unique_ptr<const char[]>* buf, Status* status) override {
    
  size_t num_entries = hash_entries_info_.entries.size();   
  
  //...
	
  // 计算所需要的的位数组大小，包含元数据
  size_t len_with_metadata = CalculateSpace(num_entries);
  std::unique_ptr<char[]> mutable_buf;
  std::unique_ptr<CacheReservationManager::CacheReservationHandle>  
    final_filter_cache_res_handle;
  // 分配内存并对齐
  len_with_metadata =       
    AllocateMaybeRounding(len_with_metadata, num_entries, &mutable_buf); 
  if (cache_res_mgr_) {     
    // 为过滤器分配的内存申请缓存资源
    Status s = cache_res_mgr_->MakeCacheReservation(         
      len_with_metadata * sizeof(char), &final_filter_cache_res_handle); 
    s.PermitUncheckedError();
  }
  
  // 计算探测次数
  int num_probes = GetNumProbes(num_entries, len_with_metadata);
  uint32_t len = static_cast<uint32_t>(len_with_metadata - kMetadataLen);  
  if (len > 0) { 
    // 将所有键的哈希值添加到位数组中  
    AddAllEntries(mutable_buf.get(), len, num_probes);  
    // 校验哈希值
    Status verify_hash_entries_checksum_status =
      MaybeVerifyHashEntriesChecksum();
  }

  bool keep_entries_for_postverify = detect_filter_construct_corruption_;
  if (!keep_entries_for_postverify) {  
    ResetEntries();
  }

  // 在位数组末尾添加元数据
  mutable_buf[len] = static_cast<char>(-1);               // 标识格式
  mutable_buf[len + 1] = static_cast<char>(0);            // 标识子实现
  mutable_buf[len + 2] = static_cast<char>(num_probes);	  // 标识探测次数

  // ...
  
  Slice rv(mutable_buf.get(), len_with_metadata); 
  *buf = std::move(mutable_buf); 
  //  统一管理分配的缓存
  final_filter_cache_res_handles_.push_back(
    std::move(final_filter_cache_res_handle));
  return rv;
}

void AddAllEntries(char* data, uint32_t len, int num_probes) {
  
  // 缓存区初始化
  constexpr size_t kBufferMask = 7;
  // 存储哈希值高32为
  std::array<uint32_t, kBufferMask + 1> hashes;
  // 存储计算出的内存偏移量
  std::array<uint32_t, kBufferMask + 1> byte_offsets;
	
  // 将哈希值填充缓冲区
  size_t i = 0;
  std::deque<uint64_t>::iterator hash_entries_it =
    hash_entries_info_.entries.begin();
  for (; i <= kBufferMask && i < num_entries; ++i) {
    uint64_t h = *hash_entries_it;
    FastLocalBloomImpl::PrepareHash(Lower32of64(h), len, data,
                                    /*out*/ &byte_offsets[i]);
    hashes[i] = Upper32of64(h);
    ++hash_entries_it;
  }

  for (; i < num_entries; ++i) {
    // 读取哈希值之后将结果写入到布隆过滤器
    uint32_t& hash_ref = hashes[i & kBufferMask];
    uint32_t& byte_offset_ref = byte_offsets[i & kBufferMask];
    FastLocalBloomImpl::AddHashPrepared(hash_ref, num_probes,
                                        data + byte_offset_ref);
    // 加载新的哈希值到缓冲区覆盖已处理的值
    uint64_t h = *hash_entries_it;
    FastLocalBloomImpl::PrepareHash(Lower32of64(h), len, data,
                                    /*out*/ &byte_offset_ref);
    hash_ref = Upper32of64(h);
    ++hash_entries_it;
  }

  // 处理剩余的哈希值
  for (i = 0; i <= kBufferMask && i < num_entries; ++i) {
    FastLocalBloomImpl::AddHashPrepared(hashes[i], num_probes,
                                        data + byte_offsets[i]);
  }
}

// h2:                  根据key计算出的hash值高32位  
// num_probes:          嗅探次数，即一个key最终映射到布隆过滤器位数组上的bit个数，也就是k  
// data_at_cache_line:  指向当前缓存行的地址 
static inline void AddHashPrepared(uint32_t h2, int num_probes,    
                                   char *data_at_cache_line) {

  uint32_t h = h2;
  // 0x9e3779b9 是一个常见的哈希常数，用于生成新的哈希值以保证探测位置的分布性
  for (int i = 0; i < num_probes; ++i, h *= uint32_t{0x9e3779b9}) {
    // 每个探测都限定在一个 512 位缓存行
    // 将哈希值的高9位提取（2^9=512bit）
    int bitpos = h >> (32 - 9);
    data_at_cache_line[bitpos >> 3] |= (uint8_t{1} << (bitpos & 7));
  }
}
```

```c++
// FastLocalBloomBitsBuilder
// 查询数据
bool FullFilterBlockReader::MayMatch(const Slice& entry,
                                     GetContext* get_context,
                                     BlockCacheLookupContext* lookup_context,
                                     const ReadOptions& read_options) const {
  
  // 加载过滤块
  CachableEntry<ParsedFullFilterBlock> filter_block;
  const Status s = GetOrReadFilterBlock(get_context, lookup_context,
                                        &filter_block, read_options);
 	//...
	
  // 获取布隆过滤器位读取器
  FilterBitsReader* const filter_bits_reader =
    filter_block.GetValue()->filter_bits_reader();
  if (filter_bits_reader) {
    // 判断是否匹配
    if (filter_bits_reader->MayMatch(entry)) {
      return true;
    } else {
      return false;
    }
  }
  return true;
}

bool MayMatch(const Slice& key) override {
  uint64_t h = GetSliceHash64(key);
  uint32_t byte_offset;
  // 预取，减少查询时的缓存未命中开销
  // 由于位数组通常存储在内存中，频繁的随机访问可能导致大量缓存未命中
  // 提前将相关内存加载到 CPU 缓存中，降低访问延迟
  FastLocalBloomImpl::PrepareHash(Lower32of64(h), len_bytes_, data_,            
                                  /*out*/ &byte_offset);
  return FastLocalBloomImpl::HashMayMatchPrepared(Upper32of64(h), num_probes_,
                                                    data_ + byte_offset);
}

// h2:               根据key计算出的hash值  
// num_probes:       嗅探次数，即一个key最终映射到布隆过滤器位数组上的bit个数 
// data_at_offset:   布隆过滤器位数组当前缓存行起始位置
static inline bool HashMayMatchPrepared(uint32_t h2, int num_probes,
                                          const char *data_at_cache_line) {
    uint32_t h = h2;
// 如果支持__AVX2__指令集（SIMD向量化运算）
#ifdef __AVX2__
    int rem_probes = num_probes;
    // 定义32位黄金比例的乘法因子
    const __m256i multipliers =
        _mm256_setr_epi32(0x00000001, 0x9e3779b9, 0xe35e67b1, 0x734297e9,
                          0x35fbe861, 0xdeb7c719, 0x448b211, 0x3459b749);

    for (;;) {
      // 将当前哈希值扩展为包含 8 个相同值的向量
      __m256i hash_vector = _mm256_set1_epi32(h);
			// 借助乘法的结合性，模拟连续乘以黄金比例常数的效果
      hash_vector = _mm256_mullo_epi32(hash_vector, multipliers);
			// 计算 9 位的位地址（即缓存行内探测位置），位地址取自 32 位值的最高 9 位
      const __m256i word_addresses = _mm256_srli_epi32(hash_vector, 28);
			
      // 从缓存行加载数据
      const __m256i *mm_data =
          reinterpret_cast<const __m256i *>(data_at_cache_line);
      __m256i lower = _mm256_loadu_si256(mm_data);
      __m256i upper = _mm256_loadu_si256(mm_data + 1);
      lower = _mm256_permutevar8x32_epi32(lower, word_addresses);
      upper = _mm256_permutevar8x32_epi32(upper, word_addresses);
      const __m256i upper_lower_selector = _mm256_srai_epi32(hash_vector, 31);
      const __m256i value_vector =
          _mm256_blendv_epi8(lower, upper, upper_lower_selector);
      // 构建掩码
      const __m256i zero_to_seven = _mm256_setr_epi32(0, 1, 2, 3, 4, 5, 6, 7);
      __m256i k_selector =
          _mm256_sub_epi32(zero_to_seven, _mm256_set1_epi32(rem_probes));
      k_selector = _mm256_srli_epi32(k_selector, 31);
      __m256i bit_addresses = _mm256_slli_epi32(hash_vector, 4);
      bit_addresses = _mm256_srli_epi32(bit_addresses, 27);
      const __m256i bit_mask = _mm256_sllv_epi32(k_selector, bit_addresses);
      // 判断探测是否匹配
      // 例如 ((~value_vector) & bit_mask) == 0)
      bool match = _mm256_testc_si256(value_vector, bit_mask) != 0;

      if (rem_probes <= 8) {
        return match;
      } else if (!match) {
        return false;
      }
      // 更新哈希值
      h *= 0xab25f4c1;
      rem_probes -= 8;
    }
// 使用简单的循环依次检查
#else
    for (int i = 0; i < num_probes; ++i, h *= uint32_t{0x9e3779b9}) {
      // 取哈希值的高9位
      int bitpos = h >> (32 - 9);
      // 判断是否匹配
      if ((data_at_cache_line[bitpos >> 3] & (char(1) << (bitpos & 7))) == 0) {
        return false;
      }
    }
    return true;
#endif
  }
```



#### Partitioned Index Filters

通常对于256MB的SST文件，通常数据块大小为4 - 32KB，而过滤器为5MB，很容易造成过滤器和数据块争夺内存空间的情况

所以RocksDB在过滤器块上添加了一个顶级索引，只有顶级索引会加载到内存中去，使用顶级索引将过滤器查询所需的分区加载到缓存中去

##### 优点：

- 更高的缓存命中率：不会让过滤器的无用块污染缓存空间，而是允许以更细的粒度加载过滤器，从而有效利用缓存空间。
- 更少的 IO 效用：只需要从磁盘加载一个分区，与读取 SST 文件的整个过滤器相比，这会减轻磁盘负载。
- 可灵活调整索引大小：不进行分区，减少过滤器内存占用的替代方法是牺牲它们的准确性，例如通过更大的数据块或更少的布隆位来分别获得更小的索引和过滤器。

##### 缺点：

- 顶级索引的额外空间：仅为索引/过滤器大小的 0.1-1％。
- 更多磁盘 IO：如果顶级索引尚未在缓存中，则会导致一次额外的 IO。为了避免这种情况，可以将它们存储在堆中或以高优先级存储在缓存中
- 失去空间局部性：如果工作负载需要频繁但随机地读取同一个 SST 文件，则会导致每次读取时加载单独的过滤器分区，这比一次读取整个过滤器效率更低。它只可能发生在 LSM 的 L0/L1 层，可以禁用分区

```c++
//添加数据
void PartitionedFilterBlockBuilder::Add(const Slice& key_without_ts) {
  AddImpl(key_without_ts, prev_key_without_ts_);
  prev_key_without_ts_.assign(key_without_ts.data(), key_without_ts.size());
}

void PartitionedFilterBlockBuilder::AddImpl(const Slice& key_without_ts,
                                            const Slice& prev_key_without_ts) {
  // 判断是否需要切分分区
  bool cut = DecideCutAFilterBlock();

  // 如果支持前缀提取器，并且当前键在前缀域中
  if (prefix_extractor() && prefix_extractor()->InDomain(key_without_ts)) {
    Slice prefix = prefix_extractor()->Transform(key_without_ts);

    if (cut) {
      // 如果需要切分分区，调用 CutAFilterBlock
      CutAFilterBlock(&key_without_ts, &prefix, prev_key_without_ts);
    }

    // 根据配置，将键和前缀添加到布隆过滤器
    if (whole_key_filtering()) {
      filter_bits_builder_->AddKeyAndAlt(key_without_ts, prefix);
    } else {
      filter_bits_builder_->AddKey(prefix);
    }
  } else {
    // 如果当前键不在前缀域中
    if (cut) {
      CutAFilterBlock(&key_without_ts, nullptr /*no prefix*/,
                      prev_key_without_ts);
    }
    // 添加完整键到布隆过滤器
    if (whole_key_filtering()) {
      filter_bits_builder_->AddKey(key_without_ts);
    }
  }
}

void PartitionedFilterBlockBuilder::CutAFilterBlock(const Slice* next_key,
                                                    const Slice* next_prefix,
                                                    const Slice& prev_key) {
  // 处理跨分区前缀的兼容性，记录下一个分区的第一个键的前缀到当前分区过滤器中
  // 例如键范围落在当前分区，但是last_key（此分区） < k < next_key（下一分区）
  // 此时查询当前分区会错误的判断不存在，k需要继续匹配下一个分区
  if (next_prefix) {
    if (whole_key_filtering()) {
      filter_bits_builder_->AddKeyAndAlt(*next_prefix, *next_prefix);
    } else {
      filter_bits_builder_->AddKey(*next_prefix);
    }
  }

  // 构建过滤器
  std::unique_ptr<const char[]> filter_data;
  Status filter_construction_status = Status::OK();
  Slice filter =
      filter_bits_builder_->Finish(&filter_data, &filter_construction_status);
  if (filter_construction_status.ok()) {
    // 验证并清空数据
    filter_construction_status = filter_bits_builder_->MaybePostVerify(filter);
  }
  
  // 构建分区过滤器唯一的索引键
  std::string ikey;
  // 启用了索引和过滤器分区
  if (decouple_from_index_partitions_) {
    if (ts_sz_ > 0) {
      // 使用上一个键 prev_key 的最小时间戳作为索引
      AppendKeyWithMinTimestamp(&ikey, prev_key, ts_sz_);
    } else {
      ikey = prev_key.ToString();
    }
    // 将分区索引键转化为内部格式
    AppendInternalKeyFooter(&ikey, /*seqno*/ 0, ValueType::kTypeDeletion);
  } else {
    ikey = p_index_builder_->GetPartitionKey();
  }
  // 将生成的过滤器数据和分区索引键保存到 filters_ 列表中
  filters_.push_back({std::move(ikey), std::move(filter_data), filter});
  partitioned_filters_construction_status_.UpdateIfOk(filter_construction_status);

  // 对于支持前缀提取的键，添加最后一个键的前缀，确保前缀查询兼容
  if (next_key && prefix_extractor() &&
      prefix_extractor()->InDomain(prev_key)) {
    filter_bits_builder_->AddKey(prefix_extractor()->Transform(prev_key));
  }
}

Status PartitionedFilterBlockBuilder::Finish(
    const BlockHandle& last_partition_block_handle, Slice* filter,
    std::unique_ptr<const char[]>* filter_owner) {
  // filters_ 是一个队列，存储了多个过滤器块
  // 队列的第一个元素是当前需要处理的过滤器
  if (finishing_front_filter_) {
    // 获取最前面的过滤器
    auto& e = filters_.front();
    
    // 编码该分区的信息
    std::string handle_encoding;
    last_partition_block_handle.EncodeTo(&handle_encoding);
    // 增量记录，计算当前块与之前块大小的差值并编码
    std::string handle_delta_encoding;
    PutVarsignedint64(
        &handle_delta_encoding,
        last_partition_block_handle.size() - last_encoded_handle_.size());
    last_encoded_handle_ = last_partition_block_handle;
    
    // 构造索引记录
    const Slice handle_delta_encoding_slice(handle_delta_encoding);
		// 添加索引记录
    index_on_filter_block_builder_.Add(e.ikey, handle_encoding,
                                       &handle_delta_encoding_slice);
    // 构造无序号的索引（只有key）
    if (!p_index_builder_->seperator_is_key_plus_seq()) {
      index_on_filter_block_builder_without_seq_.Add(
          ExtractUserKey(e.ikey), handle_encoding,
          &handle_delta_encoding_slice);
    }
		// 移除已完成的过滤器
    filters_.pop_front();
  } else {
    if (filter_bits_builder_->EstimateEntriesAdded() > 0) {
      // 如果有未完成的过滤器条目，强制完成
      CutAFilterBlock(nullptr, nullptr, prev_key_without_ts_);
    }
  }

  Status s = partitioned_filters_construction_status_;
  if (s.ok()) {
    if (UNLIKELY(filters_.empty())) {
      // filters_里面没数据，索引里面有数据，则生成一个完整的索引块
      if (!index_on_filter_block_builder_.empty()) {
        if (p_index_builder_->seperator_is_key_plus_seq()) {
          *filter = index_on_filter_block_builder_.Finish();
        } else {
          *filter = index_on_filter_block_builder_without_seq_.Finish();
        }
      } else {
        // 没有任何键被添加到过滤器
        *filter = Slice{};
      }
    } else {
      // 仍有未完成的过滤器分区需要处理，需要继续处理
      s = Status::Incomplete();
      finishing_front_filter_ = true;

      auto& e = filters_.front();
      if (filter_owner != nullptr) {
        *filter_owner = std::move(e.filter_owner);
      }
      *filter = e.filter;
    }
  }
  return s;
}

```

```c++
// 查询数据
// key:             需要检查的目标键。
// const_ikey_ptr:  指向内部键的指针，用于定位过滤器分区。
// get_context:     上下文对象，用于获取键的相关信息。
// lookup_context:  缓存查找上下文，用于优化查找操作。
// read_options:    读取选项，指定查询行为。
bool PartitionedFilterBlockReader::KeyMayMatch(
    const Slice& key, const Slice* const const_ikey_ptr,
    GetContext* get_context, BlockCacheLookupContext* lookup_context,
    const ReadOptions& read_options) {
  // 如果没启用全键过滤返回true
  if (!whole_key_filtering()) {
    return true;
  }
  return MayMatch(key, const_ikey_ptr, get_context, lookup_context,
                  read_options, &FullFilterBlockReader::KeyMayMatch);
}

bool PartitionedFilterBlockReader::MayMatch(
    const Slice& slice, const Slice* const_ikey_ptr, GetContext* get_context,
    BlockCacheLookupContext* lookup_context, const ReadOptions& read_options,
    FilterFunction filter_function) const {
  
  // 加载分区过滤器
  CachableEntry<Block_kFilterPartitionIndex> filter_block;
  Status s = GetOrReadFilterBlock(get_context, lookup_context, &filter_block,
                                  read_options);
 
  // 通过索引找到与const_ikey_ptr对应的分区过滤器，在 SST 文件中的偏移和大小
  auto filter_handle = GetFilterPartitionHandle(filter_block, *const_ikey_ptr);
  if (UNLIKELY(filter_handle.size() == 0)) {  // key is out of range
    return false;
  }

  // 从 SST 文件加载对应的分区过滤器
  CachableEntry<ParsedFullFilterBlock> filter_partition_block;
  s = GetFilterPartitionBlock(nullptr /* prefetch_buffer */, filter_handle,
                              get_context, lookup_context, read_options,
                              &filter_partition_block);
  if (UNLIKELY(!s.ok())) {
    IGNORE_STATUS_IF_ERROR(s);
    return true;
  }
	
  // 构造 FullFilterBlockReader 对象，用于访问具体的分区过滤器数据
  FullFilterBlockReader filter_partition(table(),
                                         std::move(filter_partition_block));
  // 调用传入的 FilterFunction（如 KeyMayMatch），判断键是否可能存在
  return (filter_partition.*filter_function)(slice, const_ikey_ptr, get_context,
                                             lookup_context, read_options);
}

```



