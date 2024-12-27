# RocksDB

## 概述

RocksDB 是一个高性能、嵌入式的键值存储引擎，基于 LevelDB 进行优化和扩展。它专为高并发、低延迟的读写操作设计，特别适用于大规模数据处理场景。RocksDB 采用 LSM（Log-Structured Merge-Tree）存储结构，具备高效的顺序写入性能，同时支持数据压缩、事务处理、多版本控制等功能，适合在 SSD 和持久化存储设备上运行。

## 基本结构

<img src="./images/rocksdb基本架构图.png" alt="RocksDB_introduction" style="zoom:150%;" />

## RocksDB的优缺点

### 优点

1. 插入、删除、修改操作快，具有优秀的写入性能
2. 数据冷热层分离，对于新写入、修改、删除的数据能较快的读到
3. 提供了事务支持，确保了数据在写入过程中的原子性和一致性

### 缺点

1. 由于分层的原因导致了写放大，可能造成不必要的性能损失和浪费
2. 在缓存memtable和compaction过程中对于内存消耗极大，造成性能下降和资源紧张
3. RocksDB的性能优势主要体现在写密集型和查询频率低的场景，在只读密集型应用中表现效果不佳

## 编译

```c++
git clone https://github.com/facebook/rocksdb.git
git checkout v9.9.3
注释掉makefile文件的下面语句（-Werror会要求gcc将所有warning当初error处理，导致编译不通过） 
#    WARNING_FLAGS += -Werror 
单独编译和安装方式选下面一种即可，PREFIX指定安装的位置，默认是/usr/local 
动态库    
make shared_lib    
make install-shared PREFIX=/usr/local/rocksdb 
静态库  
make static_lib    
make install-static PREFIX=/usr/local/rocksdb 
基于cmake集成到项目编译：   
rocksdb可以作为一个子项目去编译，在主目录添加自定义编译目标
add_custom_target(build-rocksdb          
		COMMAND make -j4 shared_lib -C ${ROCKSDB_SOURCE_DIR}         
    COMMENT "Building RocksDB"    
) 
编译单元测试：   
如编译db_iter_test.cc，就执行    
make db_iter_test -j4 
编译ldb：   
进入rocksdb目录   
make ldb  //默认编译debug版本    
DEBUG_LEVEL=0   make ldb  //编译release版本
```
