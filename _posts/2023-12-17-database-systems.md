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
### 虚拟内存的问题
操作系统虚拟内存(mmap)管理文件在内存和硬盘中的置换，适合只读文件。数据库一般自己管理实现定制化功能(madvice, mlock, msync)：
- 事务需要以正确的顺序flush脏页（并发控制）
- 数据预读取
- 基于语义实现更高效率的缓存置换策略，减少停顿时间
- 线程调度优化
- mmap异常处理困难：校验页, SIGBUS处理
- mmap会导致OS数据结构竞争：TLB shootdowns

### 文件表示
#### 存储管理器(存储引擎)
- 以页面形式(pages)管理数据库文件：跟踪页面读写和可用空间
- 读写调度以优化页面时间空间局部性

#### 页面page
固定大小的数据块，用于存储数据库记录。
- 可以包含元组(行)、元数据、索引、日志记录等
- 大多数数据库不混存不同类型数据
- 一些数据库要求页面内容必须自包含（页面存放描述该页内容的元数据，其它页的丢失不会影响该页内容读取）
- 每个页面需要有一个唯一标识，数据库通过间接层来映射page ids和物理存储位置(方便管理，文件位置移动只需修改page目录)
- 数据库页面大小：512B-16KB

> 硬件页：设备能够保证“故障安全写入(failsafe)”/原子写入的块

#### 页面组织：Heap file

- 无序
- API: Create/Get/Write/Delete Page
- 支持遍历所有页面
- 通过元数据跟踪已存页面和空余空间

**Page目录**
特殊的页：
- 记录每个page id到文件位置的映射
- 记录每个page空闲slots数
- 维护空闲的页面

#### 页面的布局
每个页面包含header记录了该页的元数据：
- Page size.
- Checksum.
- DBMS version.
- Transaction visibility
- Self-containment. (Some systems like Oracle require this.)

A strawman approach to laying out data is to keep track of how many tuples the DBMS has stored in a
page and then append to the end every time a new tuple is added. However, problems arise when tuples
are deleted or when tuples have variable-length attributes.
There are two main approaches to laying out data in pages: (1) slotted-pages and (2) log-structured.
Slotted Pages: Page maps slots to offsets.
- Most common approach used in DBMSs today.
- Header keeps track of the number of used slots, the offset of the starting location of the last used
slot, and a slot array, which keeps track of the location of the start of each tuple.
- To add a tuple, the slot array will grow from the beginning to the end, and the data of the tuples will
grow from end to the beginning. The page is considered full when the slot array and the tuple data
meet.
Log-Structured: 