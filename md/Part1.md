---
title: 大数据之Hbase Part1
date: 2021-07-26
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "大数据" #分类
tags:   #标签
    - Hbase
    - 大数据
---
##### Hbase的特性
>   强一致性读写: HBase 不是 "最终一致性(eventually consistent)" 数据存储. 
    这让它很适合高速计数聚合类任务。
    自动分片(Automatic sharding): HBase 表通过region分布在集群中。
    数据增长时，region会自动分割并重新分布。
    RegionServer 自动故障转移
    Hadoop/HDFS 集成: HBase 支持本机外HDFS 作为它的分布式文件系统。
    MapReduce: HBase 通过MapReduce支持大并发处理， HBase 可以同时做源和目标.
    Java 客户端 API: HBase 支持易于使用的 Java API 进行编程访问.
    Thrift/REST API: HBase 也支持Thrift 和 REST 作为非Java 前端.
    Block Cache 和 Bloom Filters: 对于大容量查询优化， 
    HBase支持 Block Cache 和 Bloom Filters。
    运维管理: HBase提供内置网页用于运维视角和JMX 度量.
##### Hbase的感性认识
hbase实操练习 可以看官方文档，先操作一遍，增强感性的认识
在HBase中，namespace命名空间指对一组表的逻辑分组，类似RDBMS中的database，方便对表在业务上划分。
help 'namespace' 可以打出来各种帮助命令，方便使用 还有help 'ddl',help 'dml'等命令。
Hbase一行数据可以类比RDBMS中的一行，表示一个实体，一个实体的所有信息都可以放到一行中。
在hbase中，操作的时候对每一个列操作，所以多次操作的时候，指定同一个rowkey,表示的是同一个对象。
需求：创建一个表，并写入一条记录，name:student.evan,age:30,phone:130xxx
在Hbase中操作如下。
``` bash
create_namespace 'evan' #创建一个自己用的namespace
list_namespace  #列出来查看一下
create 'evan:test1', 'cf1' #在这个namespace下创建一个表，只有一个列族 cf1
put    'evan:test1', 'row1', 'cf1:name', 'student.evan' #给表evan:test1插入一条数据，行键row1,列名为name
put    'evan:test1', 'row1', 'cf1:age', '30' #给表evan:test1插入一条数据，行键row1,列名为name,值30
put    'evan:test1', 'row1', 'cf1:phone', '130xxx' #给表evan:test1插入一条数据，行键row1,列名为name,值130xxx
scan   'evan:test1' #查看表中的数据
```
##### 逻辑模型和物理模型
**逻辑模型**
Hbase是google开放的论文BigTable的开源实现，它原定的使命是用来存储网络爬虫抓取的网站数据，用于搜索引擎使用。
它表中的每一行可以有不同的列，它不需要指定列，需要提前指定列族。它存储的数据没有数据类型，都是存储成字节数组。
行有行键，每一行的行键是唯一的，相同行键的插入操作被认为是同一行的操作。
列中的值可以有多个版本，有时间戳来标识不同的版本。
可以把Hbase理解为无限嵌套的HashMap。
**物理模型**
Hbase的物理存储是按列族存放的，一个列族的数据会被同一个Region管理，物理上放在一起。
空白的Cell在物理上是不存储的，返回是空值。
##### 数据模型的重要概念
Hbase中原生不支持join,它是列存储的，可以把大量的信息存储到一个表，传统的数据库表设计规范不适用于它。
Hbase中的表相对比较少，一个表中可以包括原有RDBMS中多个表的数据，通常是大宽表。
可以把它理解为kv数据库。
Hbase没有列定义，没有字段类型，这就是它被称为无模式数据库的原因。
**行键(rowkey)**
行键是不可分割的字节数组，是按字典顺序由低到高存储在表中的。
为了高效检索数据，需要仔细设计rowkey以提高查询性能，避免热点分布。
**列族**
列族是列的集合，创建表的时候只需要先定义好列族。在写入数据的时候指定列即可。
在物理上，一个列族的成员在文件系统上是存储在一起的，存储优化是针对列族级别的。
创建表的时候，至少需要指定一个列族，新的列族可以后面动态加入，但需要先disable表。
不要创建过多的列族，因为跨列族的访问是非常低效的。因为是不同的物理存储位置。
**单元格(cell)**
单元格由rowkey,column family,col,timestamp唯一确定。它的内容是不可分割的字节数组。
每一个单元格都保存着同一份数据的不同版本，不同时间版本按照时间顺序倒序排序（最新在最新面）。
TTL(time to live),用于设置单元格的生存周期，如果过期，则会被删除。
早于指定TTL值的数据会在下一次大合并时被删除。
##### 访问方式
1. 可以使用hbase shell客户端访问。
2. 如果hbase rest网关启动了，就可以使用curl 命令来访问rest api进行CRUD。
3. 使用Java api /python api/ mapreduce来访问。
4. 使用phoenix 写sql来查询hbase
   Phoenix查询引擎会将sql查询转换成一个或多个hbase scan,并行执行以生成标准的
   jdbc结果集。它支持DDL,DML,紧跟ANSI sql标准。
5. Hbase可以整合为hive的表，用hive sql来访问
``` sql
    CREATE TABLE hbase_table_1( key int, value string)
    STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
    WITH SERDEPROPERTIES (
        "hbase.columns.mapping"="cf:c1",
        "hbase.table.name"="hbase_table_1"
    )
```
##### 整体架构
**核心组件**
客户端和zookeeper，主节点HMaster和Region节点RegionServer。
zk负责多HMaster的选举，服务器之间的状态同步。存储Hbase元数据信息，实时监控RegionSever,
存储所有Region的寻址入口。
HMaster主要负责 Table和Region的管理工作：
1. 管理用户对Table的增删改查
2. 管理RegionSever的负载均衡，调整Region分布
3. Region的重分配和迁移工作
RegionServer 主要负责响应客户的IO请求，向文件系统读写数据。
**目录表(Catalog Tables)**
目录表 -ROOT- 和 .META. 作为 HBase 表存在。他们被HBase shell的 list 命令过滤掉了，
但他们和其他表一样存在。 
 -ROOT- 保存 .META. 表存在哪里的踪迹。
.META. 保存系统中所有region列表。在哪个regsion server存放。
**客户端**
HBase客户端的 HTable类负责寻找相应的RegionServers来处理行。他是先查询 .META. 和 -ROOT 目录表。
然后再确定region的位置。定位到所需要的区域后，客户端会直接去访问相应的region(不经过master)，
发起读写请求。这些信息会缓存在客户端，这样就不用每发起一个请求就去查一下。
**存储文件**
Hstore是Hbase存储的核心，它由两部分组成，memstore和StoreFile。
用户写入的数据会首先放到MemStore当中，满了以后会flush到StoreFile(Hfile),StoreFiles文件数增长到
一定的阈值，会触发Compact操作，将多个Storefiles合并成一个StoreFile,在Major Compact中会进行版本
合并和数据删除。由此可知，Hbase只增加数据，所有的更新和删除都是在Compact中进行的，这保证了它的高速写。
单个StoreFile大小达到一定阈值后，会触发Split操作，同时会把当前Region分裂成两个Region,父Region下线，
新分裂的两个子Region会被分配到相应的HRgionServer上。
**Hlog和数据恢复**
每一个RegionServer中都有一个Hlog对象，Hlog对象是一个实现了WAL的类，用户的操作在写入memStore的同时，
会写入到Hlog文件中，它会定期滚动出新，删除旧的文件。
当RegionServer意外中止后，Hmaster会通过zk感知到，首先处理遗留的Hlog文件，将其中不同Region的Log数据
进行拆分，分别放到相应的Region目录下，然后再将失效的Region重新分配。
领取到这些Region的RegionServer在加载Region的过程中，会发现有历史Hlog需要处理，会将Hlog中的数据回放到
MemStore中，然后flush到StoreFile,完成数据恢复。
