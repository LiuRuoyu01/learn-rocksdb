## WAl文件

RocksDB 的每个更新都会写入两个位置：

1. 内存中的数据结构memtable（稍后将刷新到SST文件）
2. 在磁盘上预写日志（WAL）。

WAL是记录__写操作日志__的文件。它的主要功能是确保在数据库发生故障（如宕机或崩溃）时，可以通过日志恢复未刷入 SST 文件的数据，从而避免数据丢失，**单个 WAL 捕获所有列族的写入日志**。

``` 
./ldb dump_wal --walfile=my_rocksdb/000031.log --header --print_value
打印如下 
Sequence,Count,ByteSize,Offset,Physical  Key(s) : value     
1,       1,    25,      0,      PUT(0) : 0x6B657932 : 0x76616C756532 
2,       1,    25,      32,     PUT(0) : 0x6B657933 : 0x76616C756534 
以第一行为例：序列号:1，当前操作的个数:1，字节数:25，物理偏移:0，PUT(0)表示写入操作，写入列族id为0，即默认列族，后面分别是key的值和value的值，0x6B657932转为ascii即key2，0x6B657932为value2
```

