# Local Cache

本文介绍 Local Cache 的原理，以及如何开启 Local Cache 加速外部数据查询。

在数据湖分析场景中，StarRocks 作为 OLAP 查询引擎需要扫描 HDFS 或对象存储（下文简称为“外部存储系统”）上的数据文件。查询实际读取的文件数量越多，I/O 开销也就越大。此外，在即席查询 (ad-hoc) 场景中，如果频繁访问相同数据，还会带来重复的 I/O 开销。

为了进一步提升该场景下的查询性能，StarRocks 2.5 版本开始提供 Local Cache 功能。通过将外部存储系统的原始数据按照一定策略切分成多个 block 后，缓存至 StarRocks 的本地 BE 节点，从而避免重复的远端数据拉取开销，实现热点数据查询分析性能的进一步提升。Local Cache 仅在使用外部表（不含 JDBC 外部表）和使用 External Catalog 查询外部存储系统中的数据时生效，在查询 StarRocks 原生表时不生效。

## 原理

StarRocks 将远端存储文件缓存至本地 BE 节点时，会将原始文件按照一定策略切分为相等大小的 block。Block 是数据缓存的最小单元。例如，查询 Amazon S3 上一个 128 MB 的 Parquet 文件，StarRocks 会按照默认 1 MB 的步长，将该文件拆分成相等的 128 个 block，即 [0, 1 MB)、[1 MB, 2 MB)、[2 MB, 3 MB) ... [127 MB, 128 MB)，并为每个 block 分配一个全局唯一 ID，即 cache key。Cache key 由三部分组成。

```Plain
hash(filename) + filesize + blockId
```

说明如下。

| **组成项** | **说明**                                                     |
| ---------- | ------------------------------------------------------------ |
| filename   | 数据文件名称。                                               |
| filesize   | 数据文件大小，默认 1 MB。                                    |
| blockId    | StarRocks 在拆分数据文件时为每个 block 分配的 ID。该 ID 在一个文件下是唯一的，非全局唯一。 |

假如该查询命中了 [1 MB, 2 MB) 这个 block，那么：

1. StarRocks 检查缓存中是否存在该 block。
2. 如存在，则从缓存中读取该 block；如不存在，则从 Amazon S3 远端读取该 block 并将其缓存在 BE 上。

StarRocks 默认会缓存从外部存储系统读取的数据。如果只想读取，不缓存，可进行如下设置。

```SQL
SET enable_populate_block_cache = false;
```

关于 `enable_populate_block_cache` 的更多信息，参见 [系统变量](../reference/System_variable.md#支持的变量)。

## 缓存介质

StarRocks 默认以 BE 节点的机器内存作为缓存的存储介质，也支持同时使用内存和磁盘作为两级的混合存储介质。如果您的磁盘类型是 NVMe 或 SSD 等，可同时使用内存和磁盘进行缓存；如果磁盘类型为云磁盘，例如 AWS EBS，建议使用纯内存来缓存。

## 缓存淘汰机制

在 Local Cache 中，StarRocks 采用 [LRU](https://baike.baidu.com/item/LRU/1269842) (least recently used) 策略来缓存和淘汰数据，大致如下：

- 优先从内存读取数据，如果在内存中没有找到再从磁盘上读取。从磁盘上读取的数据，会被加载到内存中。
- 从内存中淘汰的数据，会写入磁盘；从磁盘上淘汰的数据，会被废弃。

## 开启 Local Cache

Local Cache 默认关闭。如要启用，则需要在 FE 和 BE 中同时进行如下配置。

### FE 配置

支持使用以下方式在 FE 中开启 Local Cache：

- 按需在单个会话中开启 Local Cache。

  ```SQL
  SET enable_scan_block_cache = true;
  ```

- 为当前所有会话开启全局 Local Cache。

  ```SQL
  SET GLOBAL enable_scan_block_cache = true;
  ```

### BE 配置

在每个 BE 的 **conf/be.conf** 文件中增加如下参数。添加后，需重启每个 BE 让配置生效。

| **参数**               | **说明**                                                     |
| ---------------------- | ------------------------------------------------------------ |
| block_cache_enable     | 是否启用 Local Cache。<ul><li>`true`：启用。</li><li>`false`：不启用，为默认值。</li></ul> 如要启用，设置该参数值为 `true`。|
| block_cache_disk_path  | 磁盘路径。支持添加多个路径，多个路径之间使用分号(;) 隔开。建议 BE 机器有几个磁盘即添加几个路径。配置路径后，StarRocks 会自动创建名为 **cachelib_data** 的文件用于缓存 block。 |
| block_cache_meta_path  | Block 的元数据存储目录，可自定义。推荐创建在 **$STARROCKS_HOME** 路径下。 |
| block_cache_mem_size   | 内存缓存数据量的上限，单位：字节。默认值为 `2147483648`，即 2 GB。推荐将该参数值最低设置成 20 GB。如在开启 Local Cache 期间，存在大量从磁盘读取数据的情况，可考虑调大该参数。 |
| block_cache_disk_size  | 单个磁盘缓存数据量的上限，单位：字节。举例：在 `block_cache_disk_path` 中配置了 2 个磁盘，并设置 `block_cache_disk_size` 参数值为 `21474836480`，即 20 GB，那么最多可缓存 40 GB 的磁盘数据。默认值为 `0`，即仅使用内存作为缓存介质，不使用磁盘。 |

示例如下：

```Plain
# 开启 Local Cache。
block_cache_enable = true  

# 设置磁盘路径，假设 BE 机器有两块磁盘。
block_cache_disk_path = /home/disk1/sr/dla_cache_data/;/home/disk2/sr/dla_cache_data/ 

# 设置元数据存储目录。
block_cache_meta_path = /home/disk1/sr/dla_cache_meta/ 

# 设置内存缓存数据量的上限为 2 GB。
block_cache_mem_size = 2147483648

# 设置单个磁盘缓存数据量的上限为 1.2 TB。
block_cache_disk_size = 1288490188800
```

## 查看 Local Cache 命中情况

您可以在 query profile 里观测当前 query 的 cache 命中情况。观测下述三个指标查看 Local Cache 的命中情况：

- `BlockCacheReadBytes`：从内存和磁盘中读取的数据量。
- `BlockCacheWriteBytes`：从外部存储系统加载到内存和磁盘的数据量。
- `BytesRead`：从外部存储系统直接读取的数据量。

示例一：StarRocks 从外部存储系统中读取了大量的数据 (7.65 GB)，从内存和磁盘中读取的数据量 (518.73 MB) 较少，即代表 Local Cache 命中较少。

```Plain
 - Table: lineorder
 - BlockCacheReadBytes: 518.73 MB
   - __MAX_OF_BlockCacheReadBytes: 4.73 MB
   - __MIN_OF_BlockCacheReadBytes: 16.00 KB
 - BlockCacheReadCounter: 684
   - __MAX_OF_BlockCacheReadCounter: 4
   - __MIN_OF_BlockCacheReadCounter: 0
 - BlockCacheReadTimer: 737.357us
 - BlockCacheWriteBytes: 7.65 GB
   - __MAX_OF_BlockCacheWriteBytes: 64.39 MB
   - __MIN_OF_BlockCacheWriteBytes: 0.00 
 - BlockCacheWriteCounter: 7.887K (7887)
   - __MAX_OF_BlockCacheWriteCounter: 65
   - __MIN_OF_BlockCacheWriteCounter: 0
 - BlockCacheWriteTimer: 23.467ms
   - __MAX_OF_BlockCacheWriteTimer: 62.280ms
   - __MIN_OF_BlockCacheWriteTimer: 0ns
 - BufferUnplugCount: 15
   - __MAX_OF_BufferUnplugCount: 2
   - __MIN_OF_BufferUnplugCount: 0
 - BytesRead: 7.65 GB
   - __MAX_OF_BytesRead: 64.39 MB
   - __MIN_OF_BytesRead: 0.00
```

示例二：StarRocks 从外部存储系统直接读取的数据量为 0，即代表 Local Cache 完全命中。

```Plain
- Table: lineorder
 - BlockCacheReadBytes: 7.37 GB
   - __MAX_OF_BlockCacheReadBytes: 60.73 MB
   - __MIN_OF_BlockCacheReadBytes: 16.00 KB
 - BlockCacheReadCounter: 8.571K (8571)
   - __MAX_OF_BlockCacheReadCounter: 68
   - __MIN_OF_BlockCacheReadCounter: 1
 - BlockCacheReadTimer: 174.362ms
   - __MAX_OF_BlockCacheReadTimer: 753.745ms
   - __MIN_OF_BlockCacheReadTimer: 15.840us
 - BlockCacheWriteBytes: 0.00 
 - BlockCacheWriteCounter: 0
 - BlockCacheWriteTimer: 0ns
 - BufferUnplugCount: 103
   - __MAX_OF_BufferUnplugCount: 5
   - __MIN_OF_BufferUnplugCount: 0
 - BytesRead: 0.00 
```
