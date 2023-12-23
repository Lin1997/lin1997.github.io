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
![Log Structured索引](/assets/posts/log-structured-compaction.png)
- 通过[Sorted String Tables](http://ddia.vonng.com/#/ch3?id=sstables%e5%92%8clsm%e6%a0%91)(SSTables)维护id有序索引加速查找
![Log Structured索引](http://ddia.vonng.com/img/fig3-5.png)
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
比如, [Parquet将row group大小设置为128MB]([https://youtu.be/1j8SdS7s_NY?t=705]):
![Parquet PAX Organization](/assets/posts/parquet-pax-organization.png)

## Compression
### Naive Compression

### Columnar Compression