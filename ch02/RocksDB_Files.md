## 文件概述

```
//插入一些数据
(base) ➜  tools git:(v9.9.3) ./ldb --db=my_rocksdb --create_if_missing put key1 value1
...

(base) ➜  tools git:(v9.9.3) cd my_rocksdb
(base) ➜  my_rocksdb git:(v9.9.3) ll
total 228K
//sst文件，磁盘中存储kv数据的文件
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

此章节将详细介绍RocksDB的文件，其中包含了存在内存的MemTable，存在磁盘中的WAL、Manifest、SST文件

