# learn-rocksdb

由于工作需要，便开始学习RocksDB源码，并记录自己的学习过程。将学习文档分享出来供大家参考，希望对大家能有一些帮助

## 目录

- [前言](https://github.com/LiuRuoyu01/learn-rocksdb/blob/main/introduction.md)
- [1. 简介](./ch01/RocksDB_Introduction.md)
- [2. 主要文件介绍](./ch02)
  - [2.1. 文件概览](./ch02/RocksDB_Files.md)
  - [2.2. WAL](./ch02/RocksDB_WAL.md)
  - [2.3. MemTable](/ch02/RocksDB_MemTable.md)
  - [2.4. Manifest](/ch02/RocksDB_Manifest.md)
  - [2.5. SST](/ch02/RocksDB_SST.md)
- [3. 主要功能块介绍](./ch03)
  - [3.1. 布隆过滤器](./ch03/RocksDB_BloomFilter.md)
  - [3.2. 缓存](./ch03/RocksDB_Cache.md)
  - [3.3.版本](./ch03/RocksDB_Version.md)
- [4. 流程介绍](./ch04)
  - [4.1. 写流程](./ch04/RocksDB_Write.md)
  - [4.2. 读流程](./ch04/RocksDB_Read.md)
  


## 说明

由于本人水平有限，文中可能出现一些纰漏或错误的地方，欢迎大家以提交 [issue](https://github.com/lry22221111/learn-rocksdb/issues) 或 [PR](https://github.com/lry22221111/learn-rocksdb/pulls) 的方式进行更正和完善。如果文中有些描述不清晰，或者你有任何疑问和建议，都可以在 [issue](https://github.com/lry22221111/learn-rocksdb/issues) 中告诉我。