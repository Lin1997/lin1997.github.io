---
title: 数据库系统笔记
tags: 
    - 数据库
---

> **参考**
>
> - [CMU 15-445课程主页](https://15445.courses.cs.cmu.edu/fall2023/schedule.html)
> - [CMU 15-445中文字幕视频](https://www.zhihu.com/zvideo/1416127715578032128)


## 存储

### Disk-Oriented DBMS
面向磁盘的DBMS将数据存放于非易失性磁盘，操作数据前需将数据从磁盘读入内存。
- 通过Buffer Pool来管理数据在内存和磁盘之间的置换
- Execution Engine执行查询，向Buffer Pool请求指定page的数据，后者负责将page读入内存返回内存指针
- Buffer Pool Manager需保证该page在Execution Engine执行期间保持在该内存中
![面向磁盘的DBMS架构](/assets/posts/disk-oriented-dbms-overview.png)

### 虚拟内存的问题
为什么不直接以来操作系统虚拟内存?
操作系统虚拟内存(mmap)管理文件在内存和硬盘中的置换，适合只读文件。数据库一般自己管理实现定制化功能(madvice, mlock, msync)：
- 事务需要以正确的顺序flush脏页（并发控制）
- 数据预读取
- 基于语义实现更高效率的缓存置换策略，减少停顿时间
- 线程调度优化
- mmap异常处理困难：校验页, SIGBUS处理
- mmap会导致OS数据结构竞争：TLB shootdowns

### 文件表示
#### 存储管理器(存储引擎)
- 以pages形式管理数据库文件：跟踪页面读写和可用空间
- 读写调度以优化page时间空间局部性

#### Page
固定大小的数据块，用于存储数据库记录。
- 可以包含元组(行)、元数据、索引、日志记录等
- 大多数数据库不混存不同类型数据
- 一些数据库要求页面内容必须自包含（页面存放描述该页内容的元数据，其它页的丢失不会影响该页内容读取）
- 每个页面需要有一个唯一标识，数据库通过间接层来映射page ids和物理存储位置(方便管理，文件位置移动只需修改Page Directory)
- 数据库页面大小：512B-16KB

> 硬件页：设备能够保证“故障安全写入(failsafe)”/原子写入的块

#### Heap file
- 一组无序的page集合
- API: Create/Get/Write/Delete Page
- 支持遍历所有页面
- 通过元数据跟踪已存页面和空余空间

**Page Directory**
特殊的Page：
- 记录每个page id到文件位置的映射
- 记录每个page空闲slots数
- 维护空闲的页面
- 确保Page Directory和数据Page间同步(通过日志解决两种页无法原子写入)
![Page Directory数据结构](/assets/posts/heap-file-page-directory.png)


#### Page的布局
每个Page面包含header记录了该页的元数据：
- Page size.
- Checksum.
- DBMS version.
- Transaction visibility
- Self-containment. ([自包含元数据](#页面page))

Page内部如何存储数据
- [Slotted Pages Storage](#Slotted-Pages-Storage)
- [Log Structured Storage](#Log-Structured-Storage)

##### Slotted Pages Storage
多数DBMS使用的方法
- slot数组: 映射每个元组(Tuple)的起始偏移量。
- Header记录已使用slot数，最后一个slot的偏移量
![Slotted Pages布局](/assets/posts/slotted-pages-layout.png)
- 添加tupe时, slot数组从头部向尾部增长, 而tupe数据则从尾部向头部增长，两个部分重叠时标志page已满
![Slotted Pages增长](/assets/posts/slotted-pages-grow.png)
- 通过slot数组的间接映射，page内部可实现系统上层无感地移动Tuple，只需修改对应slo数组偏移量即可，上层依旧能通过同样的page id + slot数组下标查找，如一些数据库会在Tuple删除后执行compact操作。

缺点
- 碎片化：删除tuples会在page中产生gap
- 随机IO访问: 更新不同位置的tuples性能差
- 多余的IO: 更新tuples需要读取整块磁盘page

**Index-Organized Storage**
- Page存储索引数据结构
- 将Tuple作为value存储至索引数据结构中
- 通常根据key的排序存储
- 如MySQL的聚簇索引将tuple存储在B+树的叶子结点中
![Index-Organized Storage](/assets/posts/index-organized%20-storage.png)

**Record Id**
- 标识Tuple的唯一id
- 通常为: file_id + page_id + offset/slot
- 可能包含文件位置的时机

##### Log Structured Storage
- Page中存储Log Records: 记录了Tuple的修改信息
- 每个Log Entry表示了一个Tuple的Put/Delete操作, 且包含Tuple的唯一标识
- Put Record包含了Tuple内容, Delete Record将Tuple标记为已删除
- 利用磁盘顺序访问速度快的特性: 在内存中append Log Records, 顺序地写入磁盘, 已落盘page不可修改
![Log Structured追加](/assets/posts/log-structured-append.png)
- 但读取时需要从新到旧扫描Log Records
- 通过构建index，将Tuple映射到最新Log Record偏移量
![Log Structured索引](/assets/posts/log-structured-read.png)
- 可以周期性compact, 使Tuple只保留一条记录
![Log Structured Compact](/assets/posts/log-structured-compaction.png)
- 通过[Sorted String Tables](http://ddia.vonng.com/#/ch3?id=sstables%e5%92%8clsm%e6%a0%91)(SSTables)维护id有序索引加速查找
![SSTable](/assets/posts/dbms-sstable.png)
- 分层并维护多个SSTables即[Log-Structured Merge-Tree](http://ddia.vonng.com/#/ch3?id=sstables%e5%92%8clsm%e6%a0%91)(LSM tree)
- Write-Amplification(写入放大): compact时需要从磁盘读出, 合并后再次写入

#### Tuple的布局

```
Tuple:
| Header | Attribute Data |
```

- Header记录元数据如：可见信息、NULL值的BitMap等
- Data部分存放属性的实际值(以字节序列形式)，通常按创建table时指定的顺序存储
- [Catalogs](#Catalogs)包含table的schema信息, 以此来解释Attribute Data的类型和值
- 多数DBMS不允许Tuple大小超过一个page，因为要额外的元数据和指针来表示剩余部分所在overflow page
- 可能有反范式化Tuple数据，如prejoin会将不同表的相关的Tuples存储到同个page。可读取降低IO但是更新成本高

**字长对齐**
- 为了性能: 减少跨缓存行
- 减少错误: 有些ISA不支持非对齐访问([如ARM早期版本](https://medium.com/@iLevex/the-curious-case-of-unaligned-access-on-arm-5dd0ebe24965))
对齐方式有：
- 添加padding字节
- 重排序tuple的属性凑整字节

**数据类型表示**
- 变长类型如VARCHAR/VARBINARY/TEXT/BLOB: header记录长度, 跟着实际数据字节序列(大数据则存储指向External File的指针)
- 时间日期类型如TIME/DATE/TIMESTAMP: 32/64位integer表示的Unix Epoch时间戳
- Null Data Types: 
    - NULL Column Bitmap Header: 常用, bitmap指明哪些属性为NULL
    - Special Value: 如用`INT_32MIN`来代表NULL
    - 各属性都记录一个NULL Flag: 由于字节对齐可能会耗费超过1bit空间
- 定点数类型如NUMERIC/DECIMAL: 通常以精确的变长二进制形式(如string) + 额外的元数据(如长度和小数点位置)存储，如[PostgreSQL](https://github.com/postgres/postgres/blob/94415b04ed0d1f3334c413924e367b25df57f2fd/src/interfaces/ecpg/include/pgtypes_numeric.h#L18)和[MySQL](https://github.com/mysql/mysql-server/blob/19feac3674e2ae254b3e7fa7e116fc64ac284d8d/include/decimal.h#L51)。以性能换准确性, 如[PostgreSQL](https://github.com/postgres/postgres/blob/94415b04ed0d1f3334c413924e367b25df57f2fd/src/interfaces/ecpg/pgtypeslib/numeric.c#L637)
和[MySQL](https://github.com/mysql/mysql-server/blob/19feac3674e2ae254b3e7fa7e116fc64ac284d8d/strings/decimal.cc#L1840)

### Catalogs
Catalogs存放了数据库的元数据:
- Tables, columns, indexes, views
- Users, permissions
- Internal statistics

多数DBMS将Catalogs存储在特殊的tables中，如INFORMATION_SCHEMA, 用代码初始化该table


## 存储模型

### 工作负载类型
- 联机事务处理OLTP(On-Line Transaction Processing): 简单读写少量entity数据
- 联机分析处理OLAP(On-Line Analytical Processing): 读取大数据进行复杂查询
- 混合事物分析处理HTAP(Hybrid Transaction/Analytical Processing): 同时支持OLTP和OLAP
不同的存储模型适应于不同的工作负载:

### N-ary Storage Model
即Row Storage(行式存储). 将每个tuple的属性连续存储到page中
适用于OLTP: 只操作常数个entity, 大量执行insert操作.
- +insert, update, delete 很快
- +对于需要整个tuple的查询很友好
- +可以利用Index-Organized Storage
- -对于扫描大量tuple且只要其部分属性的查询效率低: 读取大量无用数据, 内存局部性差
  
  ```sql
  SELECT COUNT(U.lastLogin),
      EXTRACT(month FROM U.lastLogin) AS month
  FROM useracct AS U
  WHERE U.hostname LIKE '%.gov'
  GROUP BY EXTRACT(month FROM U.lastLogin)
  ```
  
  ![NSM执行OLAP](/assets/posts/nsm-olap-example.png)
- -不利于压缩: 单个tupe各属性值域不同压缩率低

### Decomposition Storage Model
即Column Storage(列式存储). 将所有tuple中一列属性连续存储到page中
适用于OLAP: 只读扫描整个table的部分属性
![DSM执行OLAP](/assets/posts/dsm-olap-example.png)
- +减少不必要的IO, 只读取需要的属性
- +提高了内存局部性和缓存命中率
- +更利于数据压缩
- -对于点查询, inserts, updates, deletes慢, 因为Tuple得进行splitting拆分/stitching拼接/reorganization重组

Tuple拆分和拼接(关联同个Tuple的不同属性):
- Fixed-length Offsets: 同一个属性, 值的长度相同. 同个tuple不同列值的offset相同, 该offset = 内存偏移地址 / 属性固定长度. 通过dictionary compression将变长数据转换为定长值.
![DSM Fixed-length Offsets](/assets/posts/dsm-fixed-length-offsets.png)
- 内置tuple id: 列的每一个值都额外保存一个tuple id; 还需要一个表来映射tuple id到所有属性值的位置.
![Embedded Tuple Ids](/assets/posts/dsm-embedded-tuple-Ids.png)

### Partition Attributes Across
混合了两种模式, 为了能获益于DSM的性能同时保持NSM的空间局部性: 前面OLAP查询例子中WHERE如果需要扫描多个属性, DSM的拆分会导致局部性不佳.
先将table的rows水平分割成rows group, 每个row group的属性再垂直切割成按列存储.
全局header中directory维护每个row group在文件中的offset.
![PAX Organization](/assets/posts/pax-organization.png)

比如, [Parquet](https://youtu.be/1j8SdS7s_NY?t=705)将row group大小设置为128MB:
![Parquet PAX Organization](/assets/posts/parquet-pax-organization.png)

## 压缩
- 压缩能增加每次IO获取的数据量, 减少DRAM要求
- 权衡计算速度 vs 压缩率

目标
- 生成定长值(变长的属性存到另外的pool)
- 执行查询时尽量推迟解压缩
- 无损

粒度
- Block Level: Tuple中的块
- Tuple Level: 整个tuple (适用于NSM)
- Attribute Level: 一个tuple中的一或多个属性
- Columnar Level: 多个tuple的相同属性(适用于DSM)

### Naive Compression
朴素地使用通用压缩算法以二进制形式压缩, 如gzip, LZO, LZ4, Snappy, Brotli, Oracle
OZIP, Zstd.
工程上通常不会用太高压缩率的算法保证压缩/解压速度.
朴素压缩没有考虑schema的高级语义, 无法对查询计划进行优化.
[MySQL Innodb](https://dev.mysql.com/doc/refman/8.0/en/innodb-compression-internals.html)使用了该方式:
- 将Page划分对齐为若干KB, 以减少每次读写时压缩解压的范围, 存储到Buffer Pool
- 为了避免每次写都进行解压, 将写记录到额外的mod log; 读取时记录在mod log则无序解压已压缩Page.
![MySQL Innodb Compression](/assets/posts/mysql-innodb-compression.png)

### Columnar Compression
我们希望DBMS能够无需解压, 通过将查询转换, 直接操作已压缩数据. 可以利用以下DSM的Columnar Compression算法
**Run-Length Encoding (RLE)**
- 将列中相同的值压缩为3元组: (属性值, 开始位置, 连续个数)
![Run-Length Encoding](/assets/posts/run-length-encoding.png)
- 更适用于已排序列
- 通过转换查询直接操作已压缩数据:

```sql
SELECT isDead, COUNT(*)
 FROM users
GROUP BY isDead
-- 转化为sum(连续个数)
```

**Bit-Packing Encoding**
列中所有值都比数据类型定义的范围小, 则直接截断bit数
![Bit-Packing Encoding](/assets/posts/bit-packing-encoding.png)

**Mostly Encoding**
Bit-Packing的变种, 对于小部分超出范围无法压缩的值特别维护一张表存储.
![Mostly Encoding](/assets/posts/mostly-encoding.png)

如[Amazon Redshift](http://docs.aws.amazon.com/redshift/latest/dg/c_MostlyN_encoding.html)使用了Mostly Encoding

**Bitmap Encoding**
- 为属性的每个唯一value存储一个bitmap, 某value的bitmap的第i位代表第i个tuple是否有该value, 从而无需重复地存储重复的value
- 为了避免分配大块连续内存, 可以将bitmap分割成若干block
- 仅适用于value集合的基数/势较低的属性, 否则bitmap可能比原始数据大
![Bitmap Encoding](/assets/posts/bitmap-encoding.png)

如一些数据库提供的[bitmap indexes](https://dbdb.io/browse?indexes=bitmap)

**Delta Encoding**
- 记录同一列中相邻value的差值
- 基值可以是原列中的的, 或单独存储一个查找表
- 可以结合RLE使用
![Bitmap Encoding](/assets/posts/delta-encoding.png)

**Dictionary Compression**
- 将频次高的value替换为更小的定长code, 并维护code到原始值的映射(dictionary)
- DBMS最常使用的压缩方式
- 支持快速编码/解码以及范围查询
- 支持保留与原始值相同的顺序排序
![Dictionary Compression: 保持顺序](/assets/posts/dictionary-compression-order-preserving.png)
- 优化: 如果只对压缩列进行distinct查询, 则只需读取dictionary无需扫描原列:

```sql
-- 仍然需扫描原列
SELECT name FROM users
WHERE name LIKE 'And%'

-- 只需读取dictionary
SELECT DISTINCT name
 FROM users
WHERE name LIKE 'And%'
```

Dictionary数据结构
- Array: 简单但更新成本高; 适用于不可变数据
![Dictionary的Array实现](/assets/posts/dictionary-compression-array-impl.png)
- Hash: 速度快且紧凑; 但不支持范围和前缀查询
- B+ Tree: 比hash table慢些, 且占用更多内存; 但支持范围查询和前缀查询

## 内存管理

### 目的

![面向磁盘的DBMS架构](/assets/posts/disk-oriented-dbms-overview.png)

**空间控制** 
- Page在磁盘上的写入位置
- 目标: 尽可能将经常一起使用的Page在磁盘上物理上靠近

**时间控制**
- 何时将Page读入内存以及何时将其写入磁盘
- 目标: 最小化由于从磁盘读取数据而导致的停顿次数

### Locks和Latches

**Locks**
- **高级别**的**逻辑**原语，用于保护数据库的逻辑内容(如元组、表、数据库)免受其他**事务**的干扰
- Lock将在其整个事务持续时间内持有
- 数据库系统可以向**用户公开**查询运行时持有的锁
- 锁需要能够**回滚更改**

**Latches**
- DBMS在其**内部数据结构**(如哈希表、内存区域)的临界区使用的**低级别**保护原语
- 只在执行操作的时间内持有
- 无需回滚更改

### Buffer Pool

#### 内存组织
- 在数据库内分配的一个大内存区域，用于存储从磁盘读取的Page
- 组织成一个Page的数组, 每个条目称为一个frame
- 当DBMS请求一个Page时，它会从磁盘复制到Buffer Pool的一个Frame中
- 脏页面可以被缓存起来，不立即写回
- 需要Latch来解决Page Table的线程竞争修改
![Buffer Pool组织和元数据](/assets/posts/buffer-pool-organization.png)

#### Page Table
- 一个在内存中的哈希表，用于跟踪当前在内存中的Page
- 将Page ID映射到Buffer Pool中的Frame位置; 由于Buffer Pool中Page的顺序不一定和磁盘上的顺序一致，需要该间接层来确定Page位置
- 维护每个Page的额外元数据:
  - 脏标志: 由线程在修改Page时设置, 向存储管理器指示Page必须写回磁盘
  - 引用计数: 跟踪当前访问该Page的线程数(读取或修改), 大于零则不允许存储管理器从内存中逐出该Page
  - 访问跟踪信息

#### 内存分配策略

**全局策略**

为所有活动事务找到分配内存的最佳决策

**本地策略**

- 使单个查询或事务运行更快的决策, 不考虑并发的事务
- 但需要支持page共享

大多数系统使用全局和本地视图的组合

### 优化

**Multiple Buffer Pools**

DBMS可以为不同的目的维护多个Buffer Pool, 如:
- 每个数据库一个Buffer Pool
- 每个Page类型一个Buffer Pool

每个Buffer Pool都可以采用适合其存储数据的策略; 有助于减少Latch争用并提高局部性。

将Page映射到Buffer Pool的方法:
- Object ID: 扩展Record ID以包含Object ID, 通过其将对象映射到特定Buffer Pool
![Object ID映射到Buffer Pool](/assets/posts/multiple-buffer-pools-with-object-id.png)
- Hash: 对PageID进行哈希以选择Buffer Pool。

**Pre-fetching**

通过基于查询计划, 进行Page预取来进行优化:
- 在处理第一组Page时，预取第二组Page
![顺序scan时进行Page预取](/assets/posts/dbms-page-prefetch.png)
- 根据树索引预取叶子Page

```sql
SELECT * FROM A
WHERE val BETWEEN 100 AND 250
```

![根据索引进行Page预取](/assets/posts/dbms-page-prefetch-by-index.png)

**Scan Sharing (Synchronized Scans)**

查询游标可以重用从存储中检索的数据, 或完成计算的操作数据:
- 允许多个查询连接到扫描表的单个游标
- 如果一个新查询将开始扫描表，发现存在现有查询正在执行此操作，那么DBMS会将新查询的游标连接到现有游标。DBMS跟踪新查询加入现有查询时的位置，以便在到达数据结构末尾时继续完成扫描。

**Buffer Pool Bypass**

不将顺序扫描操作获取的Page存储到Buffer Pool中以避免污染, 因为这些页面再次命中率低:
- 使用本查询的本地内存
- 也会用于需要存储临时数据的操作, 如sorting, joins

例子: Informix的[Light Scans](https://www.ibm.com/docs/en/informix-servers/14.10?topic=io-light-scans)

### 缓冲替换策略

在需要内存空间时决定从Buffer Pool中驱逐哪些Page以腾出空间。

目标
- 正确性: 还未使用完成的page不能被淘汰
- 准确性: 淘汰的page应该是未来不太会被用到的
- 速度: 算法足够高效
- 元数据开销: 减少内存开销

**最近最少使用（LRU）**

- DBMS选择驱逐具有最旧上次访问时间的Page
- 可以使用list排序, 每次访问取出并重新插入头部
![LRU List](/assets/posts/dbms-buffer-replace-lru.png)

**CLOCK**

LRU的一种近似, 无需精确地维护Page的访问顺序
- 每个Page都有一个引用位: 当访问Page时，将其设置为1。
- 将Page组织成带有“时钟手”的循环缓冲区
- 在扫描时，检查Page的位是否设置为1。如果是，则将其设置为零；如果不是，则将其淘汰。
![CLOCK](/assets/posts/dbms-buffer-replace-clock.png)

**顺序泛滥问题**

LRU和CLOCK易受到sequential flooding(顺序泛滥)的影响: 由于顺序扫描而使Buffer Pool的内容受污染, 这些页面再次命中率低。
因为LRU和CLOCK只跟踪上次访问时间, 而不是页面使用的频次

![Sequential Flooding导致Q3需要的Page0被淘汰](/assets/posts/dbms-buffer-sequential-flooding.png)
<center>Sequential Flooding导致Q3需要的Page0被淘汰</center>

解决方案:
- LRU-K: 跟踪最近K次引用的历史时间戳，并计算连续访问之间的间隔。淘汰掉间隔最大的Page。
  MySQL的近似LRU-K: 
  - 为LRU List维护两个HEAD, Young Head和Old Head, 将List划分为Young List和Old List.
  - 新Page先插入Old Head;
  - Old List里的Page被再次访问时, 将其插入到Young Head
  
  ![MySQL Approximate LRU-K](/assets/posts/mysql-approximate-lru-k.png)

- 查询的本地化: DBMS在每个事务/查询基础上选择淘汰哪些Page。这减少了每个查询对Buffer Pool的污染。
- 优先级提示: 事务根据查询执行的上下文, 告诉Buffer Pool哪些Page是重要/不重要。比如:
  - 将tree index的root所在page设置高优先级
  - 自增id插入时将覆盖id范围的page设置高优先级

### 脏页淘汰

处理带有脏标志的Page的方法:
- 最快的方式: 丢弃**不**带脏标志的任何Page
- 较慢的方式: 将脏页写回磁盘，确保其更改被持久化。

需根据硬件性能权衡: 快速淘汰干净Page, 还是将不会再次读取的脏Page写回。
通过定期后台写入脏页任务可以取消脏标志。

### 其他内存池

DBMS需要内存来存储除了Tuple和索引之外的其他信息。根据具体实现, 这些内存池可能并不总是需要写入磁盘:
- 排序 + 连接缓冲区
- 查询缓存
- 维护缓冲区
- 日志缓冲区
- 字典缓存

### 磁盘IO调度

多数DBMS文档推荐将OS调度器设置为Deadline或NOOP(FIFO), 并自己维护内部队列在更高语义上跟踪来自整个系统的页面读写请求, 以优化IO性能。如: [Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/23/ladbi/setting-the-disk-io-scheduler-on-linux.html#GUID-B59FCEFB-20F9-4E64-8155-7A61B38D8CDF), [Vertica](https://docs.vertica.com/23.3.x/en/setup/set-up-on-premises/before-you-install/manually-configured-os-settings/io-scheduling/), [MySQL](https://dev.mysql.com/doc/refman/8.0/en/innodb-linux-native-aio.html)

任务的优先级取决于多个因素：
- 顺序 vs 随机IO
- 关键任务 vs 后台任务
- 表 vs 索引(有latch故高优先级) vs 日志(尽快写出) vs 临时数据
- 事务信息
- 基于用户的服务水平协议（SLAs）

### OS Page Cache

通常系统会维护自身文件页缓存，这会导致一份数据分别在操作系统和 DBMS 中被缓存两次。大多数DBMS都会使用 (O_DIRECT) 来告诉 OS 不要缓存这些数据

## DBMS内部数据结构

DBMS在系统内部的许多不同部分使用各种数据结构。比如：
- 内部元数据：这是用于跟踪有关数据库和系统状态的信息的数据。例如: Page Tables, Page Directory
- 核心数据存储：用来存储Tuples的数据结构, 如: 聚簇索引(Hash Tables, B+ Trees)
- 临时数据结构：DBMS可以在处理查询时即时构建临时数据结构，以加快执行速度（例如，连接操作的Hash Tables）
- 表索引：可用于更快地查找特定的Tuples的辅助数据结构

在实现DBMS的数据结构时，有两个主要的设计决策需要考虑：
- 数据组织：我们需要确定如何布局内存以及在数据结构内存储哪些信息，以支持高效的访问
- 并发性：我们还需要考虑如何支持多个线程访问数据结构而不引起问题，确保数据保持正确和可靠

### Hash Tables

哈希表: 将键映射到值的无序数组
- 时间复杂度: 平均 O(1) , 最坏 O(n)。在实际应用中需考虑常数因子的优化
- 空间复杂度: O(n)

哈希表的实现包括两个部分：
- 哈希函数：如何将一个大的键空间映射到较小的域。它用于计算数组桶或槽的索引。需要权衡: 快速执行 vs 冲突率。在一个极端情况下，有一个始终返回常数的哈希函数（非常快，但存在冲突）。在另一个极端情况下，有一个“完美”的哈希函数，没有冲突，但计算时间极长。理想的设计通常位于两者之间。
- 哈希方案：在哈希后如何处理键冲突。需要权衡: 分配一个大的哈希表以减少冲突 vs 在发生冲突时执行额外指令

### 哈希函数

哈希函数接受任何键作为输入，返回该键的整数表示。该函数的输出是确定性的（即，相同的键应始终生成相同的哈希输出）。
DBMS关注哈希函数的速度和冲突率, 比较好的选择是[Facebook XXHash3](https://xxhash.com/)

### 静态哈希方案

一种哈希表大小固定的方案。这意味着如果DBMS在哈希表中的存储空间用尽，那么它必须从头开始重新构建一个更大的哈希表，这是非常昂贵的。通常，新的哈希表的大小是原始哈希表大小的两倍。
为了减少不必要的比较次数，避免哈希键的冲突是很重要的。通常，我们使用预期元素数量的两倍作为槽位的数量。
通常以下假设在现实中不成立：
- 元素数量是预先已知的。
- 键是唯一的。
- 存在一个完美的哈希函数。
因此，我们需要适当选择哈希函数和哈希方案。

#### 线性探测哈希

使用一个循环数组槽。插入发生冲突时，我们线性搜索相邻的槽，直到找到一个空槽。
对于查找操作，我们可以检查键散列到的槽，并线性搜索直到找到所需的条目。如果我们达到了一个空槽或者遍历了哈希表中的每个槽，那么该键不在表中。这意味着我们必须在槽中存储键和值，以便我们可以检查条目是否是所需的条目。
删除操作更加棘手，因为这可能会导致之后无法找到被删条目后面的条目。解决方法：
- 最常见的方法是使用“墓碑”。不删除条目，而是将其替换为一个“墓碑”条目，提示未来的查找继续扫描。
- 在删除条目后移动相邻的数据以填充现在的空槽(但要保证后续线性探测操作能正常找到)。对相邻键重新进行哈希直到找到第一个空槽, 将该键移动到该空槽。由于在键很多的情况下这样做非常昂贵，实际中很少这样实现

**处理非唯一键**

在同一个键可能与多个不同值或元组关联的情况下，有两种实现方式：
- 链表：不将值与键一起存储，而是存储一个指向包含所有值的链表的单独存储区的指针，该链表可能溢出到多个页面
- 冗余键：更常见的方法是在表中简单地多次存储相同的键。不会影响线性探测的所有操作

**优化技巧**

实现中有如下优化技巧(参考[ClickHouse](https://clickhouse.com/blog/hash-tables-in-clickhouse-and-zero-cost-abstractions))：
- 基于键的数据类型或大小的选用专门的哈希表实现，以优化存储数据、执行拆分等方面。例如，对于字符串键，我们可以将较小的字符串存储在原始哈希表中，而对于较大的字符串仅存储指针或哈希值。
- 存储额外的元数据：例如将空槽/墓碑信息打包成位图存储到页面头部，或者存储到另外的哈希表中，这将帮助避免查找已删除的键。
- 为 Table 和 Slot 维护版本元数据：由于为哈希表分配内存是昂贵的，我们可能希望复用内存。但为了清除旧表中的条目，我们可以增加Table的版本计数器。如果槽的版本与表的版本不匹配，可以将该槽视为空。而无需逐项标记每个槽为删除/空

Google 的 [absl::flat_hash_map](https://abseil.io/tips/136) 是线性探测哈希的优秀实现。

#### 布谷鸟哈希

使用多个哈希函数(或种子值)来计算记录的槽位。

> 名字源于布谷鸟在别的鸟巢中下蛋，并将别的鸟蛋挤出的行为。

在插入时，我们使用不同的哈希函数(或种子值)来寻找空槽。如果所有槽位都满，我们选择（通常是随机的）并驱逐旧条目。然后，我们将旧条目重新哈希到不同槽位的中。在罕见的情况下，我们可能会陷入循环。如果发生这种情况，我们可以使用新的哈希函数(或种子值)重建哈希表（较少见）或者使用更大的表重建哈希表（更常见）。
布谷鸟哈希保证 O(1) 的查找和删除，但插入可能更昂贵。

可以参考[libcuckoo](https://github.com/efficient/libcuckoo)的实现

### 动态哈希方案

静态哈希方案要求DBMS预设要存储的元素数量。否则，如果需要调整大小，就必须重新构建表。
动态哈希方案能够根据需要调整哈希表的大小，而无需重新构建整个表。

#### 链式哈希

这是最常见的动态哈希方案。DBMS为哈希表中的每个槽维护一个桶的链表。哈希到相同槽的键被简单地插入到该槽的链表中。
要查找一个元素，我们哈希到其桶，然后扫描查找。通过在桶指针列表中额外存储布隆过滤器，可以对此进行优化，布隆过滤器告诉我们键是否不存在于链表中，并帮助我们在这种情况下避免查找。

#### 可扩展散列

链式哈希的改进版本，它在链表无限增长之前对桶进行了分割。
重新平衡：增加哈希掩码位数，并拆分移动哈希掩码后变动了的桶条目，其他桶保持不变。
- DBMS维护掩码位数
- 当桶内已满时，DBMS会将槽数组的大小加倍以容纳新桶，并递增全局掩码位数，DBMS拆分桶并重新排列其元素。

#### 线性哈希

与在溢出时立即扩容拆分桶不同(需对Tabl Latch)，这个方案维护一个分裂指针，用于跟踪下一个要拆分的桶。
- 任何桶溢出时，在拆分指针所指向的桶(无论该指针是否指向了溢出的桶)。添加一个新的槽存放新桶指针，和一个新的哈希函数，并使用此函数对拆分桶中的键进行重新哈希
- 然后分裂指针移向下一个槽，因此分裂指针之前的槽是被分裂过的
- 插入值时，如果使用原始哈希函数将映射到曾被分裂的槽，需要应用新的哈希函数来确定键的实际位置
- 当指针到达最后一个槽时，删除上轮原始哈希函数并将指针移回到开头
- 如果拆分指针前一个桶为空，我们还可以执行反向操作回收空间：移除该桶并将拆分指针朝相反方向移动。但是这可能会导致反复的分裂和回收(震荡)