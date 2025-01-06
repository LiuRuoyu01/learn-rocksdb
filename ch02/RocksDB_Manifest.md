## Manifest文件

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


