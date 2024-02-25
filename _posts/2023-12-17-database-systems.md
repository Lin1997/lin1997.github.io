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
- 可以包含Tuple(行)、元数据、索引、日志记录等
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
- Self-containment. ([自包含元数据](#page))

Page内部如何存储数据
- [Slotted Pages Storage](#slotted-pages-storage)
- [Log Structured Storage](#log-structured-storage)

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
- 如MySQL的[聚簇索引](#聚簇索引)将tuple存储在B+树的叶子结点中
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
- [Catalogs](#catalogs)包含table的schema信息, 以此来解释Attribute Data的类型和值
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
- 每个Page类型一个Buffer Pool。如为索引创建更小Page大小的Buffer Pool: [索引节点大小](#节点大小)

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

### 替换策略

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
- 核心数据存储：用来存储Tuples的数据结构, 如: [聚簇索引](#聚簇索引)(Hash Tables, B+ Trees)
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

在同一个键可能与多个不同值或Tuple关联的情况下，有两种实现方式：
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

#### 可扩展哈希

链式哈希的改进版本，它在链表无限增长之前对桶进行了分割。
重新平衡：增加哈希掩码位数，并拆分移动哈希掩码后变动了的桶条目，其他桶保持不变。
- DBMS维护掩码位数
- 当桶内已满时，DBMS会将槽数组的大小加倍以容纳新桶，并递增全局掩码位数，DBMS拆分桶并重新排列其元素。

#### 线性哈希

与在溢出时立即扩容拆分桶不同(需对Table Latch)，这个方案维护一个分裂指针，用于跟踪下一个要拆分的桶。

![Linear Hashing](/assets/posts/dbms-linear-hashing.png)

- 任何桶溢出时，在拆分指针所指向的桶(无论该指针是否指向了溢出的桶)。添加一个新的槽存放新桶指针，和一个新的哈希函数，并使用此函数对拆分桶中的键进行重新哈希
- 然后分裂指针移向下一个槽，因此分裂指针之前的槽是被分裂过的
- 插入值时，如果使用原始哈希函数将映射到曾被分裂的槽，需要应用新的哈希函数来确定键的实际位置
- 当指针到达最后一个槽时，删除上轮原始哈希函数并将指针移回到开头
- 如果拆分指针前一个桶为空，我们还可以执行反向操作回收空间：移除该桶并将拆分指针朝相反方向移动。但是这可能会导致反复的分裂和回收(震荡)`

### 表索引

对于涉及范围扫描查询的表索引，散列表可能不是最佳选项，因为它是无序的。

表索引是表的一部分属性的副本，经过组织和/或排序以便使用其中一部分属性进行高效访问
- DBMS可以通过在表索引上进行查找来更快地找到特定的Tuple，避免执行顺序扫描
- DBMS确保表和索引的内容始终在逻辑上同步

找出执行查询时使用的最佳索引是DBMS的工作(查询优化)，数据库中创建索引的数量之间存在权衡: 
- 尽管创建更多的索引可以加快查询速度，但索引也会使用存储空间并需要维护
- 需维护索引同步, 会涉及相关并发问题

### B+树

B+树是一种自平衡树数据结构，用于保持数据排序，并允许在O(log(n))时间内进行搜索、顺序访问、插入和删除。

![B+ Tree](/assets/posts/dbms-b-plus-tree.png)

它针对读/写大块数据的面向磁盘的DBMS进行了优化:
- 原始B树在所有节点中存储键和值，而B+树仅在叶子节点中存储值，从而增加每Page存储的节点数
- 现代B+树实现结合了Blink-Tree中使用的兄弟指针(Sibling Pointers)，以支持在叶子结点间顺序遍历
- B树的每个key只在树中出现一次，空间更小；但在多线程更新时代价比B+树高，因为B+树更新时如过未破坏平衡则无需上下传播更新其它内部节点

B+树是一棵M路搜索树，具有以下特性：
- 完美平衡（即每个叶子节点都在相同的深度）。
- 除根之外的每个内部节点半满（M/2 − 1 <= 键的数量 <= M − 1）。
- 每个具有k个键的内部节点具有k+1个非空子节点。

![B+ Tree](/assets/posts/dbms-b-plus-tree-node.png)
<center>B+树节点布局：拆分建和值的数组，以提高空间局部性</center>

B+树中的每个节点都包含一个键/值对的数组:
- 对于叶子节点，键来自索引所基于的属性。尽管根据B+树的定义，每个节点上的数组并不一定需要按键排序，但实际上，几乎总是按键排序的。叶子节点值的两种方法是记录ID和Tuple数据。记录ID指向Tuple的位置，通常是主键。具有Tuple数据的叶子节点在每个节点中存储Tuple的实际内容。
- 对于内部节点，值包含指向其他节点的指针，键可以看作是导航标志。它们指导树的遍历，但不代表叶子节点上的键（因此也不代表它们的值）。这意味着你可能在内部节点中有一个在叶子节点上找不到的键（导航标志），尽管传统上内部节点仅包含在叶子节点中存在的键。
- 实现中通常拆分建和值的数组，以提高空间局部性和缓存命中率
- 根据索引类型（NULL首位还是NULL末位），空键将聚集在第一个叶子节点或最后一个叶子节点。

#### 插入

> 可视化演示: https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html

- 找到正确的叶子节点L。
- 按顺序在L中添加新条目：
   + 如果L有足够的空间，则操作完成。
   + 否则将L分裂为两个节点L和新节点L2。均匀地重新分配条目并将中间键复制到上层。在L的父节点中插入一个指向L2的条目。
- 要分裂内部节点，均匀地重新分配条目，并将中间键向上传播。

#### 删除

- 找到正确的叶子节点L。
- 删除条目：
   + 如果L至少半满，则操作完成。
   + 否则，可以尝试重新分配，从兄弟节点借用。
   + 如果重新分配失败，则合并L和兄弟节点。
- 如果发生了合并，必须删除指向L的父节点中的条目。

#### 选择条件

如果查询提供搜索键中的若干属性值，DBMS可以使用B+树索引加速查找。而哈希索引要求提供搜索键中的所有属性。
比如建立属性`<a,b>`的索引，则以下查询可以利用该索引
- 完整匹配：`(a=1 AND b=2)`
  ![索引完整匹配](/assets/posts/dbms-index-full-match.png)
- 前缀匹配：`(a=1)`
  ![索引前缀匹配](/assets/posts/dbms-index-prefix-match.png)
- Skip Scan：` (b=1)`
  ![Skip Scan](/assets/posts/dbms-index-skip-scan.png)
  在内部节点评估需要扫描的部分(a=1,b=1;a=2,b=1;a=3,b=1...)，然后多线程分散扫描，然后将结果组合到一起，如[Oracle](https://oracle-base.com/articles/9i/index-skip-scanning)和[MySQL](https://dev.mysql.com/doc/refman/8.0/en/range-optimization.html#range-access-skip-scan)

但不是所有DBMS都能完整支持。

#### 重复键

为了支持在B+树中添加重复的key，有以下方法: 
- 将Record ID附加为key的一部分。由于每个Tuple的Record ID是唯一的，这将确保所有键都不重复。
- 允许叶子节点添加溢出节点，溢出节点存放重复key。虽然没有存储冗余信息，但这种方法在维护和修改方面更为复杂。

#### 聚簇索引TODO

表按照主键指定的排序顺序进行存储，可以是堆存储或索引组织存储。
由于某些DBMS总是使用聚簇索引，如果表没有显式的主键，它们将自动创建一个隐藏的行ID作为主键；而另一些系统可能根本无法使用聚簇索引。

#### 堆聚簇TODO

Tuple在堆的页面中按照由聚簇索引指定的顺序进行排序。如果使用聚簇索引的属性访问Tuple，DBMS可以直接跳转到页面。

#### 索引扫描页面排序TODO

由于直接从非聚簇索引中检索Tuple是低效的，DBMS可以首先找出所有需要的Tuple，然后根据它们的页面ID进行排序。这样，每个页面只需要被提取一次。

### B+树设计选择

####  节点大小

根据存储介质的不同，会选用不同的节点大小：
- 存储在硬盘上的节点通常以兆字节为单位，以减少查找数据所需的寻址次数，并将昂贵的磁盘读取分摊到大块数据上
- 而内存数据库可能使用小至512字节的页面大小，以将整个页面适应到CPU缓存中，并减少数据碎片化。
这种选择也可能取决于工作负载的类型：
- 因为点查询更希望页面尽可能小，以减少加载不必要的额外信息的量
- 而大型顺序扫描可能更希望大页面以减少需要执行的读取数量。

####  合并阈值

虽然B+树有关于在删除后合并不足半满节点的规则，但实际实现违反规则可能有益，能减少调整操作次数：
- 急切地合并可能导致"抖动"，其中大量连续的删除和插入操作导致不断的拆分和合并。
- 允许批量合并，其中多个合并操作一次执行，减少树上需要进行的昂贵写锁的时间。
有一种合并策略会保留树中的小节点，并稍后重建它，使得树不平衡（如Postgres中的情况）TODO

####  变长键

目前我们只讨论了具有固定长度键的B+树。然而，我们可能还想支持变长键，例如，大键的小子集导致了大量浪费的空间。有几种处理这种情况的方法：
- 指针 / Record ID
  不直接存储键，而是只存储指向键的指针。由于对每个键都要根据指针间接访问的效率较低，实际生产中只有嵌入式设备使用这种方法以节省空间。
- 变长节点
  将节点设计为支持变长大小。由于处理变长节点的内存管理开销很大，这通常是不可行的。
- 填充
  将每个键的大小设置为最大键的大小，并填充所有较短的键。这是对内存的巨大浪费，故没有人使用这种方法。
- 键映射 / 间接映射
  节点内维护映射字典，内容为原键的前缀 + 指向键值数据的指针。这是最常用的方法，节省空间，并且可能快捷地执行点查询（因为索引指向的键-值对与叶子节点指向的完全相同）。还允许某些索引搜索和叶子扫描甚至无需追踪指针（如果前缀与搜索键已经不匹配）。
  ![Key Map / Indirection](/assets/posts/dbms-index-key-indirection.png)

#### 节点内搜索

找到一个节点后，还需要在节点内搜索（无论是从内部节点找到下一个节点，还是在叶子节点中找到我们的键值）。这需要考虑一些权衡：
- 线性: 最简单的解决方案是扫描节点中的每个键，直到找到我们的键。
  - +无无需对键进行排序，从而使插入和删除更加迅速。
  - -相对低效，每次搜索O(n)的复杂度。可以使用 SIMD（或等效）指令对其进行矢量化。
  ![使用SMID指令进行矢量化线型查找](/assets/posts/dbms-intra-node-linear-search-by-simd.png)
  使用SMID指令进行矢量化线型查找: [_mm_cmpeq_epi32_mask](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpeq_epi32&ig_expand=862)
- 二分:
  用于搜索的更有效解决方案是保持每个节点排序，并使用二分搜索找到键。
  - +搜索效率更高，每次搜索O(ln(n))的复杂度。
  - -必须维护每个节点的排序，插入变得更加昂贵。
- 插值: 在某些情况下，我们可能能够利用插值来找到键。这种方法需将节点键的分布信息存储为元数据（例如最大元素，最小元素，平均值等），并使用它估计键的近似位置。
  例如，如果我们正在查找一个节点中的8，并且我们知道10是最大键，10 - (n + 1)是最小键（其中n是每个节点中的键数），那么我们知道可以从最大键前第2个插槽开始向前搜索，因为在这种情况下，离最大键一个插槽的键必须是9。
  - +最快的方法
  - -由于其对具有某些属性的键（如整数）和复杂性的适用性有限，这种方法仅在学术数据库中可见。TODO

### 优化

#### 指针交换

因为B+Tree的每个节点都存储在Buffer Pool的Page中，每次加载新Page时，我们需要从Buffer Pool中获取它，这需要latching和查找。
为了跳过这一步，我们可以用原始指针代替Page ID（称为"swizzling"）。可以趁着正常遍历索引时，从Page查找结果获取原始指针并替换Page ID。
请注意，我们必须持续跟踪哪些指针已经进行了"swizzle"和"deswizzle"，并在它们指向的Page被unpinned和淘汰时将它们反向转换回Page ID。
这对于靠近root的结点很有用。

#### 批量插入

当首次构建B+Tree时，使用通常的方式插入每个键将导致不断的分割操作。由于我们已经给予叶子节点兄弟指针，因此如果我们构建一个叶子节点的排序链表，然后使用每个叶子节点的第一个键轻松地从下向上构建索引，那么初始插入数据的效率将更高。请注意，根据我们的上下文，我们可能希望尽可能紧密地打包叶子节点以节省空间，或者在每个叶子节点中留出空间，以允许在需要分割之前插入更多数据。

#### 前缀压缩

当在同一节点中具有相同前缀的键时，通常会存在某些键的部分重叠（因为相似的键在B+Tree中排序后会相邻排列）。我们可以通过将前缀存储到节点开头，然后只在每个插槽中包含每个键的唯一部分来避免多次存储相同前缀。
![索引节点前缀压缩](/assets/posts/dbms-index-prefix-compression.png)

#### 重复消除

在允许非唯一键的索引的情况下，我们可能会发现叶子节点反复包含相同的键，但附有不同的值。对此的一个优化可能是只写入键一次，然后跟随其所有关联的值。

#### 后缀截断

在大多数情况下，内部节点中的键仅用作导航标志，而非其实际数据键值（即使键存在于索引中，我们仍然必须搜索至叶子结点以确保它没有被删除）。因此在内部节点无需存储完整键，只存储足够正确路由到正确叶子节点所需的最小前缀。
![索引节点后缀截断](/assets/posts/dbms-index-suffix-truncation.png)

#### 优化写入的B+树

分割/合并节点操作很昂贵。因此，一些B-Tree的变体，例如Bϵ-Tree，在内部节点中记录更改，并延迟将更新传播到叶节点。


## 索引并发控制

### 索引并发控制
到目前为止，我们假设我们讨论的数据结构是单线程的。然而，大多数数据库管理系统（DBMS）需要允许多个线程安全地访问数据结构，以利用额外的CPU核心并隐藏磁盘I/O延迟。

有些系统使用单线程模型。将单线程数据结构转换为多线程的一种简单方法是使用read-write lock，但这并不高效。

并发控制协议是DBMS用来确保对共享对象的并发操作得到“正确”结果的方法。

协议的正确性标准可能有所不同：
- 逻辑正确性：这意味着线程能够读取它应该期望读取的值，例如，线程应该能够读取它之前写入的值。
- 物理正确性：这意味着对象的内部表示是健全的，例如，数据结构中没有指针会导致线程读取无效的内存位置。

本讲只关心强制执行物理正确性。我们将在后续讨论逻辑正确性。

### Locks vs Latches
在讨论DBMS如何保护其内部元素时，Locks和Latches之间有一个重要的区别。

#### Locks
Locks是一种高级的逻辑原语，它保护数据库内容（例如，元组、表、数据库）免受其他事务的干扰。事务在其整个持续时间内都会持有Locks。数据库系统可以向用户展示正在运行的查询所持有的Locks。应该有一些高级机制来检测死锁并回滚更改。

#### Latches
Latches是用于保护DBMS内部数据结构（例如，数据结构、内存区域）的低级保护原语。Latches在数据库系统中用于简单操作（如: page latch）时仅持有很短的时间。Latches有两种模式：
- 读：允许多个线程同时读取同一个项目。即使另一个线程已经以读模式获取了Latches，线程也可以获取读模式的Latches。
- 写：只允许一个线程访问项目。如果另一个线程以任何模式持有Latches，线程无法获取写Latches。持有写Latches的线程也会阻止其他线程获取读Latches。


### Latches的实现
Latches的实现应该具有较小的内存占用，并且在没有竞争时可能有快速路径来获取Latches。
用于实现Latches的基本原语是通过现代CPU提供的原子指令。通过这些指令，线程可以检查内存位置的内容，以确定它是否具有某个值。

在DBMS中有几种实现Latches的方法。每种方法在工程复杂性和运行时性能方面都有不同的权衡。这些测试和设置步骤是原子执行的（即，在测试和设置步骤之间，没有其他线程可以更新值）。

#### Test-and-Set Spin Latch(TAS)
Spin Latch是操作系统mutex的更有效替代方案，因为它由DBMS控制。Spin Latch本质上是线程尝试更新的内存位置（例如，将布尔值设置为true）。线程执行CAS操作，尝试更新内存位置。DBMS可以控制无法获取Latches时的行为。它可以选择再次尝试（例如，使用while循环），或允许操作系统将其取消调度。因此，这种方法比操作系统mutex给了DBMS更多的控制，因为在无法获取Latches时，后者的控制权交给了操作系统。

- 示例：std::atomic<T>
- 优点：latch/unlatch操作效率高（在x86上只需单指令）。
- 缺点：不可扩展且对缓存不友好，因为多个线程会在不同线程中多次执行CAS指令。这些浪费的指令会在高竞争环境中累积；线程在操作系统看来很忙，尽管它们没有进行有用的工作。这会导致缓存一致性问题，因为线程在其他CPU上轮询缓存行。

#### Blocking OS Mutex
Latches的一种可能实现是操作系统内置的mutex基础设施。Linux提供了futex（fast user-space mutex），它由（1）用户空间的Spin Latch和（2）操作系统级别的mutex组成。如果DBMS能够获取用户空间latch，那么latch就会被设置。即使它包含两个内部latches，对DBMS来说它看起来像是一个单一的latch。如果DBMS无法获取用户空间latch，那么它会进入内核并尝试获取更昂贵的mutex。如果DBMS无法获取这个第二个mutex，那么线程会通知操作系统它被阻塞在mutex上，然后它会被取消调度。

操作系统mutex通常在DBMS内部不是个好主意，因为它由操作系统管理，并且开销很大。

- 示例：std::mutex
- 优点：使用简单，不需要在DBMS中进行额外编码。
- 缺点：昂贵且不可扩展（每次latch/unlatch调用大约25纳秒），因为操作系统调度。

#### Reader-Writer Latches
Mutex和Spin Latch不区分读/写（即，它们不支持不同模式）。DBMS需要一种方法来允许并发读取，这样在应用程序读取量很大时，性能会更好，因为读者可以共享资源，而不是等待。

Reader-Writer Latches允许Latches以读或写模式持有。它跟踪每种模式下持有Latches的线程数量以及等待获取每种模式Latches的线程数量。不同的DBMS可以有不同的策略来处理队列。

值得注意的是，不同的实现有不同的等待策略。有读者优先、写者优先和公平的Reader-Writer Latches。在不同的操作系统和pthread实现中，行为有所不同。

- 示例：std::shared_mutex
- 优点：允许并发读取。
- 缺点：DBMS必须管理读写队列以避免饥饿。由于额外的元数据，存储开销比Spin Latch大。


### Hash Table Latching
由于线程访问数据结构的方式有限，因此在静态哈希表中支持并发访问是容易的。例如，当从一个Slot移动到下一个Slot时，所有线程都以相同的方向移动（即自顶向下）。线程也一次只访问一个Page/Slot。因此，在这种情况下不可能发生死锁，因为不可能有两个线程竞争对方持有的Latches。当我们需要调整表大小时，我们只需对整个表获取一个全局Latches来进行操作。

在动态哈希方案（如[可扩展哈希](#可扩展哈希)）中，Latches的实现更为复杂，因为有更多共享状态需要更新，但总体方法相同。

支持哈希表Latches的两种方法在粒度上分为：

- **Page Latches**：
  每个Page都有自己的读写Latches，保护其全部内容。线程在访问Page之前获取读或写Latches。这降低了并行性，因为可能一次只有一个线程可以访问Page，但对于单个线程来说，访问Page中的多个Slot将非常快速，因为它只需要获取一个Latches。

- **Slot Latches**：
  每个Slot都有自己的Latches。这增加了并行性，因为两个线程可以在同一Page上访问不同的Slot。但它增加了存储和计算开销，因为线程必须为它们访问的每个Slot获取Latches，每个Slot都必须存储Latches数据。DBMS可以使用单模式Latches（即，Spin Latch）来减少元数据和计算开销，代价是牺牲一些并行性。

还可以直接使用CAS指令创建一个无Latches的线性探测哈希表。通过尝试将特殊“空”值与我们希望插入的元组进行比较和交换，可以在Slot中插入。如果失败，我们可以探测下一个Slot，直到成功。


### B+树Latches

B+树Latches存的挑战在于防止以下两个问题：
- 线程同时尝试修改节点的内容。
- 一个线程在遍历树的同时，另一个线程正在分裂/合并节点。

Latch crabbing/coupling协议允许多个线程同时访问/修改B+树。基本思想如下：
1. 获取父节点的Latches。
2. 获取子节点的Latches。
3. 如果子节点被认为是“安全的”，则释放父节点的Latches。所谓“安全”的节点是指在更新时不会发生分裂、合并或重新分配的节点。换句话说，节点是“安全”的，如果
  - 对于插入：它不是满的。
  - 对于删除：它的填充度超过一半。

请注意，读Latches不需要担心“安全”条件。

#### Basic Latch Crabbing Protocol
- **搜索**：从根节点开始向下，重复获取子节点的Latches，然后释放父节点的Latches。
- **插入/删除**：从根节点开始向下，根据需要获取X Latches。一旦子节点被latched，检查它是否安全。如果子节点安全，释放所有祖先节点的Latches。

从正确性的角度来看，释放Latches的顺序并不重要。然而，从性能的角度来看，最好释放树中较高位置的Latches，因为它们会阻塞对更多叶子节点的访问。

#### Improved Latch Crabbing Protocol
Basic Latch Crabbing Protocol的问题在于，事务总是在每次插入/删除操作时获取根节点的独占Latches。这限制了并行性。相反，我们可以假设需要调整大小（即，分裂/合并节点）的情况很少，因此事务可以获取到叶子节点的共享Latches。并使用读Latches和抓取来到达它并进行每个事务都会假设到达目标叶子节点的路径是安全的，验证。如果叶子节点不安全，那么我们就中止并使用Basic Latch Crabbing Protocol重新开始事务。

- **搜索**：与之前相同的算法。
- **插入/删除**：像搜索一样设置读Latches，到达叶子节点，并在叶子节点上设置写Latches。如果叶子节点不安全，释放所有先前的Latches，并使用Basic Latch Crabbing Protocol重新开始事务。

#### 叶子节点扫描
上述协议中的线程以“自顶向下”的方式获取Latches。这意味着线程只能从当前节点下方的节点获取Latches。如果所需的Latches不可用，线程必须等待直到它变得可用。鉴于此，永远不可能发生死锁。

然而，叶子节点扫描容易受到死锁的影响，因为现在我们有线程同时尝试以两个不同方向获取Latch且其中有独占Latch（例如，线程1尝试删除，线程2进行叶子节点扫描）。索引Latches不支持死锁检测或避免。

因此，程序员处理这个问题的唯一方法是通过编码规范。叶子节点兄弟Latches获取协议必须支持“no-wait”模式。也就是说，B+树代码必须能够处理失败的Latches获取。由于Latches旨在被（相对）短暂持有，如果线程尝试获取叶子节点的Latches，但该Latches不可用，那么它应该快速中止其操作（释放它持有的任何Latches），并重新启动操作。

## 排序与聚合算法

### 查询计划
数据库系统将SQL编译成查询计划: 操作符组成的树结构.
数据从叶子流向根节点.
根节点的输出即为查询结果.
```sql
SELECT R.id, S.cdate
 FROM R JOIN S
 ON R.id = S.id
WHERE S.value > 100
```
![查询计划](/assets/posts/dbms-query-plan.png)

对于面向磁盘的数据库系统，查询结果不一定能完全放在内存中。
- 我们将使用buffer pool来实现需要溢写(spill)到磁盘的算法。
- 我们希望最小化算法的I/O操作, 最大化顺序I/O.

### 排序
- 在关系模型下，表中的元组无顺序
- 排序可能用于：ORDER BY、GROUP BY(详见[聚合](#聚合))、DISTINCT(消除重复)和JOIN操作符
- 如果需要排序的数据能够完全装入内存，那么DBMS可以使用标准的排序算法（例如，快速排序）
- 如果数据太大而无法装入内存，那么DBMS需要使用能够按需溢写到磁盘的外部排序，并且优先考虑顺序I/O而不是随机I/O(故快速排序不太合适)

#### Top-N Heap Sort
如果查询包含带有LIMIT的ORDER BY，那么DBMS使用[堆排序](https://en.wikipedia.org/wiki/Heapsort)，只需要扫描一次数据就能找到前N个元素。
堆排序的理想情况是当前N个元素都能够装入内存时，这样DBMS只需要在扫描数据时维护一个内存中的排序优先队列（最大/小堆）。
```sql
SELECT * FROM enrolled
ORDER BY sid
LIMIT 4
```
![Top-N Heap Sort](/assets/posts/dbms-heap-sort.png)

#### External Merge Sort
对于无法完全装入内存的数据的标准排序算法是外部归并排序。它是一种分治排序算法，将数据集分割成多个独立的归并段(run)，然后分别对它们进行排序。它可以按需将归并段(run)溢写(spill)到磁盘，之后逐个将它们读回。该算法由两个阶段组成：

- 阶段#1 – 排序：首先，算法对能够装入主内存的小块数据进行排序，然后将排序后的page写回磁盘。
- 阶段#2 – 合并：然后，算法将排序后的子文件合并成一个更大的单一文件。

对于每个归并段的values, 有两个选择:
- 提前物化(early-materialized)，将整个tuple存储到page中
- 延迟物化(late-materialized)，我们只在排序归并段中存储记录ID，稍后再读取

#### Two-way Merge Sort
算法的最基本版本是两路归并排序。
- 排序阶段: 读取每个page(从磁盘或或拷贝已在内存page)，对其进行排序，然后将排序后的版本写回磁盘
- 合并阶段: 需使用三个buffer pool page。它从磁盘读取两个排序page，并将它们合并到第三个page中。每当第三个page填满时，它就会被写回磁盘，并用一个空page替换。DBMS通常提供配置page大小选项(排序内存/工作内存)
![Two-way Merge Sort](/assets/posts/dbms-two-way-merge-sort.png)
如果N是数据page的总数，该算法总共进行$1 + ⌈\log_2 N⌉$次数据遍历（第一次用于排序阶段，然后是$⌈\log_2 N⌉$次用于递归合并阶段）。
总的I/O成本是2N × （遍历次数），因为每次遍历都需要对每个page进行一次I/O读取和一次I/O写回。

#### General (K-way) Merge Sort
算法的泛化版本允许DBMS利用多于三个buffer pool page。
设B是可用的page总数，在排序阶段，算法可以一次读取B个page，并将⌈N/B⌉个排序归并段写回磁盘。合并阶段也可以每次合并多达B-1个归并段，同样使用一个page来合并数据，并根据需要写回磁盘。

在泛化版本中，算法执行$1 + ⌈\log_B−1 ⌈N/B⌉⌉$次遍历（一次用于排序阶段，$\log_B−1 ⌈N/B⌉$次用于合并阶段）。
总的I/O成本是2N × （遍历次数），因为它在每次遍历中都需要对每个page进行一次读取和写回。

#### 双缓冲优化
外部归并排序的一个优化是预取下一个归并段并在后台存储到第二个缓冲区中，同时系统处理当前归并段。
这通过持续利用磁盘减少了每个步骤的I/O请求等待时间。这种优化需要使用多线程，因为预取应该在当前归并段的计算发生时进行。

#### 比较优化
代码特化(Code Specialization)经常用于加速排序比较。区别于将比较器作为函数指针提供给排序算法，可以将比较逻辑硬编码到特定键类型的排序方法，节省指针和函数调用等开销。C++中的模板特化就是一个例子。

另一种优化（针对基于字符串的比较）是后缀截断。对于长VARCHAR键，先尝试用其二进制前缀([如Rabin–Karp算法的滚动哈希](https://zh.wikipedia.org/wiki/旋转哈希))进行相等检查，如果前缀相等，则回退到较慢的字符串比较。

#### 使用B+树
DBMS可使用已有的B+树索引来辅助排序。
- 如果索引是聚簇索引，DBMS可以直接遍历B+树。由于索引是聚簇的，数据将以正确的顺序存储，因此I/O访问将是顺序的。这意味着它总是比外部归并排序更好，因为不需要进行计算。
- 如果索引是非聚簇的，遍历树几乎总是更糟糕的，因为每条记录可能存储在任何page上，所以几乎所有的记录访问都需要(随机)磁盘读取。

![使用B+树索引辅助排序](/assets/posts/dbms-sort-by-using-index.png)
因此优化器必须评估不同查询计划的I/O，并决定是否使用索引。

### 聚合
查询计划中的聚合操作符将一个或多个元组的值合并为单个标量值(GROUP BY, AVG，COUNT，MIN，MAX，SUM, DISTINCT)。
为了减少I/O消耗，实现聚合有两种方法：（1）排序和（2）哈希。

#### 排序聚合
如果聚合要求有序(有ORDER BY):
- DBMS首先根据GROUP BY键对元组进行排序：如果所有数据都能装入缓冲池，它可以使用内存中的排序算法（例如，快速排序），或者如果数据大小超过内存，它可以使用外部归并排序算法。
- 然后DBMS对排序后的数据进行顺序扫描以计算聚合: 
  - DISTINCT: 丢弃重复
  - GROUP BY: 执行聚合计算

```sql
SELECT DISTINCT cid
 FROM enrolled
WHERE grade IN ('B','C')
ORDER BY cid
```
![Sorting Aggregation](/assets/posts/dbms-sorting-aggregation.png)
在执行排序聚合时，重要的是要合理安排查询操作以提高效率。例如，如果查询需要过滤，最好先执行过滤(谓词/过滤下推)，然后对过滤后的数据进行排序，以减少需要排序的数据量。

#### 哈希聚合
如果聚合无需有序，哈希计算聚合通常比排序计算量更低：
DBMS在扫描表时填充一个临时哈希表。对于每条记录，检查哈希表中是否已经有条目，并进行适当的修改:
- DISTINCT: 丢弃重复
- GROUP BY: 执行聚合计算

如果哈希表的大小太大而无法装入内存，那么DBMS必须将其溢写到磁盘。完成这一过程有两个阶段：
- 阶段#1 – 分区：使用哈希函数h1根据目标哈希键将元组分区到磁盘上。这将把所有匹配的元组放入同一个分区。假设总共有B个缓冲区，我们将有B-1个输出缓冲区用于分区，1个缓冲区用于输入数据。如果任何分区已满，DBMS将将其溢写到磁盘。
- 阶段#2 – 重哈希(到内存)：对于磁盘上的每个分区，将其读入内存，并基于第二个哈希函数h2（其中h1 ≠ h2）构建内存中的哈希表(这假设每个分区都能装入内存)。然后遍历这个哈希表的每个桶，将匹配的元组集中起来以计算聚合。由于经过了阶段#1的h1相同键会在统一分区，当阶段#2中的内存哈希表无法插入新条目时可以直接丢弃旧条目，因为插入新条目说明旧条目已经完成聚合计算。

```sql
SELECT DISTINCT cid
 FROM enrolled
WHERE grade IN ('B','C')
```
![External Hashing Aggregation: Partition](/assets/posts/dbms-external-hashing-aggregation-partition.png)

![External Hashing Aggregation: Rehash](/assets/posts/dbms-external-hashing-aggregation-rehash.png)
在重哈希阶段，DBMS可以存储形式为（GroupByKey→RunningValue）的对来计算聚合。RunningValue的内容取决于聚合函数，如AVG()应为COUNT和SUM.
```sql
SELECT cid, AVG(s.gpa)
 FROM student AS s, enrolled AS e
WHERE s.sid = e.sid
GROUP BY cid
```
![Hashing Summarization](/assets/posts/dbms-hashing-summarization.png)
将新元组插入哈希表的方式：
- 如果找到匹配的GroupByKey，则相应地更新RunningValue。
- 否则插入一个新的键值对（GroupByKey→RunningValue）。
