---
id: tuning.md
---

# 性能调优

## 插入性能调优

<div class="alert note">
“数据插入”到“数据写入磁盘”的基本流程请参考 <a href="storage_operation.md">存储原理</a>。
</div>

在不超出单次插入上限（256 MB）的前提下，批量插入比单条插入要高效得多。

系统配置中的两个参数对插入性能有影响：

- `wal_config.enable`

该参数用于开启或关闭 [预写日志（WAL, Write Ahead Log）](write_ahead_log.md) 功能（默认开启）。开启和关闭预写日志功能时，插入数据的流程分别如下：

* 开启预写日志功能时，预写日志模块先将数据写入磁盘，然后返回插入操作。
* 关闭预写日志功能时，数据插入速度更快。系统直接将数据写入内存中的可写缓冲区，并立即返回插入操作。

但是对于 [删除操作](storage_operation.md#删除) 来说，打开预写日志功能时速度更快。为了保证数据的可靠性，我们建议打开预写日志。

- `auto_flush_interval`

该参数是指后台落盘任务的间隔时间，默认值为 1 秒。根据 Milvus [数据段合并策略](storage_operation.md#数据合并)，增大该值可减少段合并的次数，减少磁盘 I/O，提高插入操作的吞吐量。

<div class="alert note">
Milvus 无法搜索到在该时间间隔内未落盘的数据。
</div>

另外，建立集合时的参数 `index_file_size` 也对插入性能有影响。该参数的默认值为 1024 MB，最大值为 4096 MB。该参数越大，将文件合并到该值设定的大小所需的次数就越多，影响插入操作的吞吐量。该参数越小，则产生的数据段越多，查询性能可能会变差。

除了软件层面的因素外，网络带宽和存储介质对插入操作性能也有影响。

## 查询性能调优

影响查询性能的因素包括硬件环境、系统参数、索引、查询规模等。

### 硬件环境

- 使用 CPU 计算时，查询性能取决于 CPU 的主频、核心数和支持的指令集。

<div class="alert note">
Milvus 在支持 AVX 指令集的 CPU 上的查询性能较好。
</div>

- 使用 GPU 计算时，查询性能取决于 GPU 的并行计算能力以及传输带宽。

### 系统参数

<div class="alert note">
系统参数配置请参考 <a href="configuration.md">Milvus 服务端配置</a>。
</div>

- `cache_config.cpu_cache_capacity`

该参数是指用于驻留查询数据的缓存空间大小，默认值为 4 GB。如果该缓存空间不足以容纳所需的数据，查询时会从磁盘临时加载数据，严重影响查询性能。因此，`cpu_cache_capacity` 应当大于查询所需的数据量。

浮点型原始向量的数据量可根据“向量总条数 × 维度 × 4”来估算，二进制型原始向量的数据量可根据“向量总条数 × 维度 ÷ 8”来估算。

索引建立完成后（不包括 FLAT），索引文件需要额外占用磁盘空间，查询只需加载索引文件。

* IVFFLAT 索引的数据量和其原始向量总数据量基本相等。
* IVFSQ8 / IVFSQ8H 索引的数据量相当于其原始向量总数据量的 25% ～ 30%。
* IVFPQ 索引的数据量根据其参数变化，一般低于其原始向量总数据量的 10%。
* HNSW／RNSG／ANNOY 索引的数据量都大于其原始向量总数据量。

<div class="alert note">
通过调用 <code>get_collection_stats</code> 接口，可准确获知查询一个集合所需的数据总量。
</div>

- `gpu_config.gpu_search_threshold`

在 GPU 版本中，当目标向量数量大于等于该参数设定的值，将会启用 GPU 查询。该参数的默认值为 1000。

GPU 查询的性能取决于 CPU 将数据加载进显存的速度以及 GPU 的计算能力。在目标向量数量较少时，无法充分发挥出 GPU 并行计算的优势。只有当目标向量数量达到某个阈值后，GPU 的查询性能才会优于 CPU。在实际使用中，可根据实验对比得出该参数的理想值。

- `gpu_resource_config`

指定用于查询或建索引的 GPU 设备。对于数据插入和查询并发的场景，使用 GPU 建索引可避免后台建索引任务和查询任务争抢 CPU 资源。对于查询目标向量数量较大的场景，使用多 GPU 可显著提高查询效率。

### 索引

<div class="alert note">
向量索引的基本概念请参考 <a href="index_overview.md">向量索引概述</a>。
</div>

选择合适的索引需要在存储空间、查询性能、查询召回率等多个指标中权衡。

- FLAT 索引

FLAT 是对向量的暴力搜索（brute-force search），速度最慢，但召回率最高（100%），磁盘空间占用最小。
  
随着目标向量数量增多，使用 CPU 做 FLAT 查询的耗时呈线形上升关系；而使用 GPU 查询时，批量查询的效率高，目标向量数量增加对查询耗时影响不大。

- IVF 系列索引

IVF 系列索引包括 IVF_FLAT、IVF_SQ8／IVF_SQ8H 和 IVF_PQ。IVF_SQ8／IVF_SQ8H 和 IVF_PQ 索引对向量数据做了有损压缩，磁盘占用量较少。
  
IVF 索引都有两个相同的参数：`nlist` 和 `nprobe`，相关原理可参考 [索引介绍](index.md#索引介绍)。

根据其原理，可估算出使用 IVF 索引进行查询时的计算量。

* 单个分段计算量可估算为：目标向量数量 × (`nlist` + （段内向量数 ÷ `nlist`）× `nprobe`)
* 分段的数量可估算为：集合数据总量 ÷ `index_file_size`
* 对集合查询所需的计算总量则为：单个分段计算量 × 分段数量 

通过估算得出的计算总量越大，查询耗时越长。实际使用中可根据以上公式确定合理的参数，在满足召回率的前提下获得较高的查询性能。

<div class="alert note">
在持续插入数据的场景下，由于对大小未达到 <code>index_file_size</code> 的分段未建立索引，对其使用的查询方式是暴力搜索。计算量为：目标向量数量 x 该分段向量总数。
</div>

- HNSW / RNSG / ANNOY 索引

HNSW、RNSG、ANNOY 的索引参数对查询性能的影响较为复杂，建议参考 [索引介绍](index_type.md#索引介绍)。

### 其他

- 结果集

结果集的规模取决于目标向量数量和 `topk`。`topk` 的大小对计算的影响不大。但在目标向量数量和 `topk` 都较大的情况下，结果集序列化和网络传输的耗时会相应增加。

- MySQL

Milvus 使用 MySQL 作为元数据后端服务。Milvus 在查询数据时会多次访问 MySQL 以获取元数据信息，因此 MySQL 服务的响应速度对 Milvus 的查询性能有较大影响。

- 预加载

首次查询需要先将数据从磁盘读入缓存，因此耗时较长。为避免首次查询加载数据，可预先调用 `load_collection` 接口，或使用系统参数  `preload_collection` 指定启动 Milvus 时要预先加载的集合。

- 段数据整理

在 [数据段整理](storage_operation.md#数据段整理) 中提到，Milvus 在查询数据时将 **delete_docs** 读入内存以过滤被删除的实体。调用 `compact` 接口可清理被删除的实体，减少过滤操作，从而提高查询性能。

## 存储优化
  
- 数据段整理

在 [数据段整理](storage_operation.md#数据段整理) 中提到，被删除的实体不参与计算，并且占用磁盘空间。如果有大量的实体已被删除，你可以调用 `compact` 接口来释放磁盘空间。