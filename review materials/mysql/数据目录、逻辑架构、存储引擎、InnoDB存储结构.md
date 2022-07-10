# 数据目录

## 常用目录

* mysql默认安装目录：`/var/lib/mysql/`，查看变量：`show variables like 'datadir'`。
* 命令目录：/usr/bin和/usr/sbin；常用命令：myqladmin、mysqlbinlog、mysqldump。
* 配置文件目录：/etc/mysql/、/usr/share/mysql-8.0。

## 数据库与文件系统

### 默认数据库

* mysql：核心数据库，存储用户账户和权限、存储过程、事件定义、运行时的日志、时区、帮助。
* information_schema：存储数据库中**维护的所有数据库的信息**，比如某个数据库中的表、视图 、触发器、列、索引等。这些信息并不是真实的用户数据，而是一些描述性信息，一般称为**元数据**。此数据库中以**innodb_sys**开头的表，用于表示内部系统表。
* performance_schema：存储运行时的一些状态信息，可用于**监控MySQL服务的各类指标**。包括统计最近执行了哪些语句，在执行过程的每个阶段都花费了多长时间，内存的使用情况等信息。
* sys：MySQL 系统自带的数据库，这个数据库主要是通过**视图**的形式把 **information_schema** 和 **performance_schema** 结合起来，帮助系统管理员和开发人员监控 MySQL 的技术性能。

### 表在文件系统中的表示

在数据目录（/var/lib/mysql ）下除了information_schema 这个系统数据库外，其他的数据库在数据目录下都有对应的子目录。

#### InnoDB

* 表结构：新创建一张表，InnoDB则会在对应的数据库目录下创建一个用于存储**描述表结构的文件**，文件名：表名.frm。注意：在MySQL8.0中，不再单独提供此文件，而是合并到**表名.ibd**文件中。

* 表数据和索引

  * 系统表空间（system tablespace）：默认情况下，InnoDB会在数据目录下创建一个名为 ibdata1 （大小为 12M ）的文件，注意：这个文件是**自扩展文件** ，当大小不够用的时它会自

    己增加文件大小。

  * 独立表空间（file-per-table tablespace）：在mysql5.6.6及之后的版本，InnoDB默认不再把表的数据存储到系统表空间中，而是为**每个表创建一个独立表空间**。使用独立表空间的情况下，会在该表所属数据库目录下创建一个表示该独立表空间的文件（表数据和索引都存储在这个文件中），文件名：表名.ibd。

  * 系统表空间与独立表空间的设置：

    ```xml
    # 0：代表使用系统表空间； 1：代表使用独立表空间
    [server]
    innodb_file_per_table=0
    ```

  * 其它类型的表空间：随着MySQL的发展，除了上述两种老牌表空间之外，现在还新提出了一些不同类型的表空间，比如通用表空间（general tablespace）、临时表空间（temporary tablespace）等。

#### MyISAM

* 表结构： 每新创建一张表，MyISAM则会在对应的数据库目录下创建一个用于存储**描述表结构的文件**，文件名：表名.frm。注意：MySQL8.0中文件名为；表名.xxx.sdi。
* 表数据和索引：MyISAM的索引全部都是**二级索引**，**数据和索引**是分开存储的。文件名分别为：表名.frm（存储表结构）、表名.MYD（存储表数据MYDATA）、表名.MYI（存储表索引MYINDEX）。

### 总结

* 如果存储引擎是InnoDB，新创建一张表，则会在数据库对应的目录下创建一个名为**表名.frm**的文件，用于存储描述表结构的文件。
* 如果使用系统表空间的模式，表数据和索引都会存储在**idbdata1**文件中。
* 如果使用独立表空间的模式，表数据和索引则会存储在**表名.ibd**文件中。
* 此外：在MySQL5.7中每个数据库目录下都会生成db.opt文件，此文件用于保存数据的相关配置。比如：字符集、比较规则。而MySQL8.0不再提供此文件。

# 逻辑架构

## 逻辑架构剖析

所有的数据，数据库、表的定义，表的每一行的内容，索引，都是存在 文件系统 上，以 文件 的方式存

在的，并完成与存储引擎的交互。当

### 连接层

1. 账号密码身份校验、权限获取。
   * 用户名或密码不对，会收到一个Access denied for user错误，客户端程序结束执行。
   * 用户名密码认证通过，会从权限表查出账号拥有的权限与连接关联，之后的权限判断逻辑，都将依赖于此时读到的权限。
2. 一次连接会分配一个线程进行处理，之后的操作都会由分配到的线程进行处理。

### 服务层

* SQL Interface（SQL接口）：接收客户端的SQL命令和命令运行结果。MySQL支持DML（数据操作语言）、DDL（数据定义语言）、存储过程、视图、触发器、自定义函数等多种SQL语言接口。
* Parse（解析器)：
  * 对SQL进行语法分析、语义分析，将SQL语句分解成**数据结构**，用于传递到后续步骤，之后的操作都基于此结构。如果分解遇到错误，那么就说明SQL语法是不合理的。
  * 在SQL命令传递到解析器的时候会被解析器验证和解析，并为其创建**语法树** ，并根据数据字典丰富查询语法树，会**验证该客户端是否具有执行该查询的权限** 。
* Optimizer（查询优化器）：SQL进行解析之后，查询之前会由查询优化器确定SQL语句的执行路径，生成**执行计划**。执行计划表明查询时使用的索引，表之间的连接顺序等。
* Caches & Buffers（查询缓存）：
  * 组成：表缓存、记录缓存、key缓存、权限缓存、SQL查询缓存等。
  * 作用：对SQL语句和结果进行缓存，如果缓存中有则不会进行查询解析、优化和执行等过程就可以将结果返回给客户端。
  * 查询缓存是可以在**不同客户端之间共享**的，简单理解就是SQL语句一样即可。
  * 在MySQl5.7.20开始，不再推荐使用查询缓存，并在**MySQL8.0中删除**，因为缓存命中率太低，缓存是通过SQL语句进行缓存的，而不是通过执行计划、语法树进行缓存。

### 引擎层

插件式存储引擎层（ Storage Engines），**真正的负责了MySQL中数据的存储和提取，对物理服务器级别维护的底层数据执行操作**，服务器通过API与存储引擎进行通信。不同的存储引擎具有的功能不同，可根据实际需要进行选取。

### 存储层

数据库、表（结构、数据、索引）等都是存在**文件系统**上，以**文件**的方式存在的，并完成与存储引擎的交互。

### 总结

![image-20220703015732410](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703015732410.png)

## SQL执行流程

### 执行流程

![image-20220703020352746](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703020352746.png)

1. 查询缓存：如果查询缓存中有此SQL语句的缓存，则直接返回给客户端，如果没有则进入解析阶段。需要说明的是，查询缓存的命中率很低，在MySQL8.0中废弃此功能。因为在MySQL查询中不是缓存查询计划，而是查询对应的结果，意味着只有**相同的查询操作才会命中查询缓存**。简单理解就是两个查询请求在任何字符上的不同（如：空格、大小写、注释等）的会导致查询不会命中。如果查询中包含相同函数表、自定义变量和函数、系统表，那么此请求就不会被缓存。缓存是有会**缓存失效**的，对表的结构和数据进行修改，都会导致被修改表的缓存全部被删除。总之，缓存往往是**弊大于利**。

2. SQL解析：由解析器对SQL语句进行语法分析、语义解析，根据语法规则，判断输入的SQL语句是否满足MySQL语法。

   ![image-20220703022306222](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703022306222.png)

3. 优化器：由优化器确定SQL语句的执行路径，如：使用的索引、表的连接顺序等。在查询优化器中，可以分为**逻辑查询优化阶段**和**物理查询优化阶段**。 逻辑查询优化就是通过改变 SQL 语句的内容来使得 SQL 查询更高效，同时为物理查询优化提供更多的候选执行计划。
4. 执行器：在执行之前需要判断是否具备权限，如果没有则会返回权限错误；如果有则会执行SQL查询并返回结果。在MySQL8.0以下的版本，如果设置了查询缓存，查询到结果之后会对结果进行缓存。

5. 总结：

   SQL 语句在 MySQL 中的执行流程： SQL语句 → 查询缓存 → 解析器 → 优化器 → 执行器。

   ![image-20220703023501341](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703023501341.png)

### 执行原理

* 通过profiling查看SQL执行的流程原理

```shell
# 查看profiling是否开启
select @@profiling
show variables like 'profiling'

# 设置profiling，0：关闭 1：开启
set profiling=1

# 查看profiling，查看当前会话的所有profiles
show profiles

# 查看profile
show profile # 查看最近的一次
show profile for query 7 # 根据指定Query_ID查看
```

![image-20220703024725718](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703024725718.png)

```shell
# 查询更丰富的内容
show profile cpu,block io for query 6
```

![image-20220703024816606](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703024816606.png)

* 在MySQL5.7版本中是有查询缓存的，如果开启查询缓存，会有所不同。

  * 开启查询缓存

    ```sh
    # 修改配置文件
    query_cache_type=1
    
    # 重启mysql服务
    systemctl restart mysqld
    ```

  * 执行同一条语句多次后查看profile

    * 第一次执行的profile

      ![image-20220703030328847](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703030328847.png)

    * 第二次执行的profile

      ![image-20220703030056713](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703030056713.png)

  > 同一条SQL语句执行多次，如果开启查询缓存，第一次查询之后会把查询结果放入到缓存中，第二次查询会直接从缓存中获取并返回给客户端。

## 数据库缓冲池

### 概述

InnoDB 存储引擎是以**页**为单位来管理存储空间的，我们进行的增删改查操作其实本质上都是在访问页面（包括读页面、写页面、创建新页面等操作）。而磁盘 I/O 需要消耗的时间很多，而在内存中进行操作，效率则会高很多，为了能让数据表或者索引中的数据随时被我们所用，DBMS 会申请**占用内存来作为数据缓冲池** ，在真正访问页面之前，需要把在磁盘上的页缓存到内存中的 Buffer Pool 之后才可以访问。
这样做的好处是可以让磁盘活动最小化，从而 **减少与磁盘直接进行 I/O 的时间** 。要知道，这种策略对提升 SQL 语句的查询性能来说至关重要。如果索引的数据在缓冲池里，那么访问的成本就会降低很多。

![image-20220703155958330](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703155958330.png)

> InnoDB 缓冲池包括了数据页、索引页、插入缓冲、锁信息、自适应 Hash 和数据字典信息等。

### 特性

* 缓冲池会优先对使用频次高的热数据进行加载。因为内存的大小无法对硬盘的数据做到全部加载。
* 预读。在使用到一些数据时，大概率还会使用到周围的一些数据，因此也会把周围的一些数据加载到缓冲池中，这样可以减少未来可能的磁盘I/O操作。

### 查看/设置缓冲池

```mysql
# 查看缓冲池大小
show variables like 'innodb_buffer_pool_size';
# 设置缓冲池大小
set global innodb_buffer_pool_size = 268435456;

# 查看缓冲池实例的个数,默认为1个
show variables like 'innodb_buffer_pool_instances';
# 设置缓存池实例个数
set global innodb_buffer_pool_instances = 2;
```

> 推荐缓冲池大小最大不要超过机器内存的百分之六十，大小推荐4096的公因数倍，因为4096B是一个内存页的大小也是一个磁盘扇区的大小。
>
> 每个缓冲池实例实际占用的内存空间大小为：innodb_buffer_pool_size / innodb_buffer_pool_instances 。
>
> 注意：缓冲池实例个数不是设置的越多越好，管理缓冲池也是需要性能开销的。InnoDB规定，在innodb_buffer_pool_size小于1GB时设置多个innodb_buffer_pool_instances是无效的，InnoDB默认会自动修改为1。推荐在innodb_buffer_pool_size大于1GB时将innodb_buffer_pool_instances设置为多个。

```ini
# 修改mysql配置文件的方式设置缓冲池的大小
[server]
innodb_buffer_pool_size = 268435456

# 修改mysql配置文件的方式设置缓冲池实例的个数
innodb_buffer_pool_instances = 2
```

> tips：修改配置文件需要重启mysql服务才可以生效。

### 引申问题

* 如果执行 SQL 语句的时候更新了缓存池中的数据，那么这些数据会马上同步到磁盘上吗？
* 我们都知道更新数据时是先更新缓冲池中的数据后再同步到磁盘上的，如果在执行更新期间出现错误，导致部分数据已更新，部分数据未更新，这时候该怎么办？如果已全部更新到缓冲池中，在向磁盘同步时出现了错误，这时候该怎么办？
* 答：Redo Log & Undo Log

# 存储引擎

## 查看/设置存储引擎

### 默认存储引擎

```mysql
# 查看mysql提供的存储引擎类型
show engines;
show engines \G;

# 查看默认的存储引擎
show variables like '%storage_engine%';
# 或者
SELECT @@default_storage_engine;

# 设置默认的存储引擎
SET DEFAULT_STORAGE_ENGINE=MyISAM;
```

```ini
# 修改配置文件的方式设置默认存储引擎
default-storage-engine=MyISAM

# 重启mysql服务，修改配置文件后需重启mysql才会生效
systemctl restart mysqld.service
```

### 表的存储引擎

如果在创建表的语句中没有显式指定表的存储引擎的话，那就会默认使用**mysql服务默认存储引擎**作为表的存储引擎。

* 查看表的存储引擎

  ```mysql
  SHOW CREATE TABLE 表名\G;
  ```

* 创建表时指定存储引擎

  ```mysql
  CREATE TABLE 表名( 
      建表语句; 
  ) ENGINE = 存储引擎名称;
  ```

* 修改表的存储引擎

  ```mysql
  ALTER TABLE 表名 ENGINE = 存储引擎名称;
  ```

## 存储引擎类型

### InnoDB

* 具备外键支持功能的事务存储引擎。

* 从MySQL3.23.34a开始支持InnoDB存储引擎。在MySQL5.5之后，InnoDB为默认存储引擎。

* InnoDB是MySQL的默认事务存储引擎，它被设计用来处理大量的短期(short-lived)事务。可以确保事务的完整提交(Commit)和回滚(Rollback)。

* InnoDB在处理巨大数据量有很大的优化，因此InnoDB是**为处理巨大数据量的最大性能设计**。

* 对比MyISAM存储引擎：

  * InnoDB支持事务、行锁，MyISAM只支持表锁。

  * InnoDB在进行操作时效率比MyISAM差，因为InnoDB寻址是映射到块再到行，而MyISAM是直接到文件的OFFSET；InnoDB还要维护MVCC（多版本并发控件）一致，虽然一些场景没有，但InnoDB还是需要去检查和维护的。

  * InnoDB用于存储数据和索引的磁盘大小比MyISAM大，因为MyISAM进行了数据压缩。

  * InnoDB对内存要求较高，内存大小对性能有决定性的影响。因为MyISAM只缓存索引，不缓存真实数据；InnoDB不仅缓存索引还缓存真实数据。

    ![image-20220703175514872](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703175514872.png)

### MyISAM

* 不支持事务、行锁、外键；崩溃后无法进行数据安全恢复。
* MySQL5.5之前默认的存储引擎。
* 优势：访问速度快；对事务完整性无要求或以SELECT、INSERT为主的，推荐使用MyISAM。
* count(*)的查询效率很高，因为针对数据统计有额外的常量进行存储。

### Memory

* Memory采用的逻辑介质是**内存** ， **响应速度快** ，但是当mysqld守护进程崩溃的时候**数据会丢失** 。另外，要求存储的数据是**数据长度不变**的格式，比如，Blob和Text类型的数据不可用(长度不固定的)。
* Memory同时**支持哈希（HASH）索引** 和**B+树索引** 。
* MEMORY **表的大小是受到限制**的。表的大小主要取决于两个参数，分别是**max_rows**和**max_heap_table_size**。其中，max_rows可以在创建表时指定；max_heap_table_size的大小默认为16MB，可以按需要进行扩大。
* 数据文件和索引文件是分开存储的。表结构描述信息是存储在磁盘中数据库对应的目录下，文件名为：表名.frm ，数据存储在内存中。
* 适用的场景：目标数据量小、操作访问频繁的数据、临时的数据、立即可用的数据。

### Archive

* 基本上用于数据归档；数据压缩比非常的高，存储空间大概是innodb的十分之一和十五分之一。

* 因为不支持索引、缓存索引和数据，所以并不适合并发访问。

* Archive使用行锁来实现高并发插入操作，但是它不支持事务，其设计目标只是提供高速的插入和压缩功能。

  ![image-20220703173858363](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703173858363.png)

### Blackhole

丢弃写操作，读操作会返回空内容；作用：未知。

### CSV

* 一般用于存储Excel数据。

* 在创建CSV表时都会在数据库目录下创建**表名.CSM**和**表名.CSV**两个文件，CSM文件（元文件）存储的是表元数据信息（表的状态、表中存在的行数）；CSV文件存储的是数据，每列数据以逗号隔开。如：

  ```tex
  "1","record one" 
  "2","record two"
  ```

* CSV文件被表格应用读取，甚至写入。

### 其它存储引擎

*  Federated：Federated引擎是访问其他MySQL服务器的一个**代理** ，尽管该引擎看起来提供了一种很好的**跨服务器的灵活性** ，但也经常带来问题，因此**默认是禁用的** 。
* Merge：管理多个MyISAM表构成的表集合。
* NDB：MySQL集群专用存储引擎。也叫做 NDB Cluster 存储引擎，主要用于**MySQL Cluster 分布式集群**环境，类似于 Oracle 的 RAC 集群。

### InnoDB不为人知的秘密

* 自适应哈希索引：InnoDB会监控对表上各索引页的查询，如果观察该数据被访问的频次符合规则，那么就建立哈希索引来加快数据访问的速度，这个哈希索引称之为"**Adaptive Hash Index,AHI**",AHI是通过缓冲池的B+树页构建的，建立的速度很快，而且不对整颗树都建立哈希索引。(可以理解成热点的数据才会进入这个哈希表)

# InnoDB数据存储结构

## 存储结构

* 磁盘与内存交互的基础单位：页。

* InnoDB将数据划分为若干个页，页的默认的大小为16KB。

* 因为页是数据库管理存储空间的基本单位，也是数据库I/O操作的最小单位，所以在读取数据时，不管读取的是一行还是多行，都是将行所在的页进行加载；如果是写操作，也是会对数据所在页进行操作，向磁盘进行同步（刷盘操作）时，也是会对整个页进行同步。

* 不同的数据页可能存储在磁盘中不同的位置，可能是**不连续**的，页与也之间是以**双向链表**的方式进行关联的。每个数据页中，不同的记录会按照主键从小到大的顺序组成一个**单向链表**。每个数据页都会为页中存储的记录生成一个**页目录**，方便使用**二分法**快速定位到数据对应的槽，然后再遍历该槽对应分组中的记录即可快速找到指定的记录。

* 页大小的查看

  ```mysql
  show variables like '%innodb_page_size%';
  ```

  > 不同的数据库管理系统（简称DBMS）的页大小不同。比如在MySQL的InnoDB存储引擎中，默认页的大小是**16KB**。
  >
  > 页大小是可以进行修改的，修改变量或者修改配置文件即可；页大小建议修改为2^n。

* 在数据库中，页的上层结构还有区（Extent）、段（Segment）和表空间（TableSpace）的概念。

  ![image-20220324200502569](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220324200502569.png)

  > 1Extent = 64 * 16KB = 1MB
  >
  > 段（Segment）由一个或多个区组成，区在文件系统是一个连续分配的空间（在InnoDB中是连续的64个页)，不过在段中不要求区与区之间是相邻的。**段是数据库中的分配单位，不同类型的数据库对象以不同的段形式存在**。当创建数据表、索引的时候，就会相应创建对应的段。
  >
  > 表空间（Tablespace)是一个逻辑容器，表空间存储的对象是段，在一个表空间中可以有一个或多个段，但是一个段只能属于一个表空间。数据库由一个或多个表空间组成，表空间从管理上可以划分为系统表空间，**用户表空间、撤销表空间、临时表空间**等。

## 页的内部结构

* 页的类型分类：数据页（保存B+树节点）、系统页、Undo页和事务数据页等；数据页是最常使用的页。

* 数据页的存储空间划分为七部分（共16KB）：文件头（File Header）、页头（Page Header）、最大最小记录（Infimum + supremum)、用户记录（User Records）、空闲空间（Free Space）、页目录（Page Directory）和文件尾（File Tailer）。

  | 名称             | 占用大小 | 说明                                               |
  | ---------------- | -------- | -------------------------------------------------- |
  | File Header      | 38字节   | 文件头，描述页的信息（页的编号、其上一页、下一页） |
  | Page Header      | 56字节   | 页头，页的状态信息                                 |
  | lnfimum-Supremum | 26字节   | 最大和最小记录，这是两个虚拟的行记录               |
  | User Records     | 不确定   | 用户记录，存储行记录内容                           |
  | Free Space       | 不确定   | 空闲记录，页中还没有被使用的空间                   |
  | Page Directory   | 不确定   | 页目录，存储用户记录的相对位置                     |
  | File Trailer     | 8字节    | 文件尾，校验页是否完整                             |

* 第一部分：File Header（文件头部）、File Trailer（文件尾部）。

  * File Header：

    * FIL_PAGE_OFFSET（4字节）：页号；每个页都有一个唯一的页号。

    * FIL_PAGE_TYPE（2字节）：页的类型。

      ![image-20220703205147152](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703205147152.png)

    * FIL_PAGE_PREV（4字节）和 FIL_PAGE_NEXT（4字节）：上一页的页号和下一页的页号。

      ![image-20220703205340609](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703205340609.png)

    * FIL_PAGE_SPACE_OR_CHKSUM（4字节）：当前页校验和；文件头部和文件尾部都有这个属性；用于保证页的完整性。

    * FIL_PAGE_LSN（8字节）：页最后修改时对应的日志序列位置（LSN）。

  * File Trailer：前4个字节代表页的校验和；页最后修改时对应的日志序列位置（LSN）；用于校验页的完整性，如果文件头部和文件尾部的LSN值校验不成功的话，就说明同步过程出现了问题。

* 第二部分：Free Space（空闲空间）、User Records（用户记录）、Infimum + Supremum（最小最大记录）

  * Free Space：当存储记录时会按照指定的**行格式**存储到User Records中，在生成页的时候其实并没有User Records，而是**当每插入一条记录时从Free Space中申请此记录大小的空间到User Records中**。当Free Space的空间被User Records申请完之后，意味着页已被使用完毕，如果还有新记录插入，则需要去**申请新的页**。

    ![image-20220703210842110](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703210842110.png)

  * User Records：用于存储插入的记录；按照指定的行格式进行存储，记录与记录之间使用单链表进行关联。详细的内容请参考**行格式的记录头信息**。

  * Infimum + Supremum：最小最大记录；记录的大小是根据主键进行比较的。这两条记录并不存放在User Records中，而是单独存储在 Infimum + Supremum 中，由MySQL自动插入（称为伪记录或虚拟记录）。

    ![image-20220703211723651](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703211723651.png)

* 第三部分：Page Directory（页目录）、Page Header（页面头部）

  * Page Directory

    * 为什么需要页目录？

      答：在页中，记录是单向链表的形式进行存储的，检索效率不高，因此专门做一个目录，方便二分查找。

    * 页目录的设计：

      1. 将页中所有的记录（包括最小最大记录，不包括已删除记录）进行分组。

      2. 第一组只有最小记录；最后一组包含最大记录，数量为1-8条；其余组记录数量为4-8条之间；除第一组外，其余组的记录数量**尽量平分**。

      3. 每组中最后一条记录的**头信息**中都会存储当前组共有多少条记录，字段为**n_owned**。

      4. 页目录用来存储每组最后一条记录的**地址偏移量**，这些地址偏移量会按照先后顺序存储起来，每组的地址偏移量也被称之为**槽（slot）**，每个槽相当于指针指向了不同组的最后一个记录。

         ![image-20220703230823841](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703230823841.png)

  * Page Header：存储页中各种状态信息。

    ![image-20220703231114481](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703231114481.png)

## 行格式

### 概述

* InnoDB行格式类型：Compact（紧密）、Redundant（冗余）、Dynamic（动态）和Compressed（压缩）。

  ```mysql
  # 查询默认行格式
  select @@innodb_default_row_format;
  
  # 查询指定表的行格式
  show table status like '表名' \G
  
  # 创建表指定行格式
  CREATE TABLE 表名 (列的信息) ROW_FORMAT=行格式名称
  
  # 修改表的行格式
  ALTER TABLE 表名 ROW_FORMAT=行格式名称
  ```

* 以下内容使用**COMPACT**行格式进行详细讲解。

### COMPACT

* 变长字段列表：把记录中所有变长字段（VARCHAR、VARBINARY、TEXT，BLOB）的真实数据占用的字节长度都存放在记录的开头部位，从而形成一个变长字段长度列表。长度值按照列的逆序存放，使用16进制进行表示。

  ![image-20220703222124813](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703222124813.png)

  ![image-20220703222139811](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703222139811.png)

* NULL值列表：把记录中所有为NULL值的列进行统一管理；值为1时代表列值为NULL，值为0是代表列值不为NULL。之所以要存储NULL是因为数据都是需要对齐的，如果**没有标注出来NULL值**的位置，就有可能在查询数据的时候出现**混乱**。如果使用一个特定的符号放到相应的数据位表示空置的话，虽然能达到效果，但是这样很浪费空间，所以直接就在行数据得头部开辟出一块空间专门用来记录该行数据哪些是非空数据，哪些是空数据。

  ![image-20220703222920231](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703222920231.png)

  ![image-20220703222932448](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703222932448.png)

* 记录头信息（5字节）

  ![image-20220703223100082](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703223100082.png)

  ![image-20220703223148081](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703223148081.png)

  ![image-20220703223106896](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703223106896.png)

* delete_mask：标记着当前记录是否被删除；0未删除，1已删除。

  > 被删除的记录为什么还在页中存储呢？
  > 你以为它删除了，可它还在真实的磁盘上。这些被删除的记录之所以不立即从磁盘上移除，是因为移除它们之后其他的记录在磁盘上需要**重新排列，导致性能消耗**。所以只是打一个删除标记而已，所有被删除掉的记录都会组成一个所谓的**垃圾链表**，在这个链表中的记录占用的空间称之为**可重用空间**，之后如果有新记录插入到表中的话，可能把这些被删除的记录占用的存储空间覆盖掉。

* min_rec_mask：标记当前记录是否是B+Tree非叶子节点层中最小的记录；0否，1是。

* record_type：当前记录的类型； 0：普通记录、1：B+树非叶节点记录、2：最小记录、3：最大记录。

* heap_no：当前记录在页中的位置；最小记录和最大记录的heap_no值分别是0和1，详细参考页的**内部结构的Infimum + Supremum**。

* n_owned：标记页目录当前组中共有多少条记录，每组最后一条记录才会存储此字段；详情见页目录（page directory）。

* next_record：表示从当前记录到下一条记录的**地址偏移量**。

  ![image-20220703224925883](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703224925883.png)

  > 删除记录
  >
  > ![image-20220703225048115](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703225048115.png)
  >
  > 添加记录，将已删除的再添加回来
  >
  > ![image-20220703225151216](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703225151216.png)

### 行溢出

* InnoDB存储引擎可以将一条记录中的某些数据存储在真正的数据页面之外。

* MySQL数据库提供的VARCHAR(M)类型，真实数据是否可以存放65535个字节？

  答：否。因为除了真实的数据之外，还需要存储额外的信息，变长字段标识需要2字节，NULL值标识需要1字节。

* 页的大小一般为16KB，也就是16384个字节，而ARCHAR(M)类型的列最大可存储665533个字节，这样会出现一个页存放不了一条记录，这种现象称为**行溢出**。

* 在Compact和Reduntant行格式中，对于占用存储空间非常大的列，在记录的真实数据处只会存储该列的一部分数据，把剩余的数据分散存储在几个其他的页中进行分页存储，然后记录的真实数据处用20个字节存储指向这些页的地址（包括这些分散在其他页中数据占用的字节数），从而可以找到剩余数据所在的页。

  ![image-20220703233443184](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703233443184.png)

### Dynamic、Compressed

* Dynamic：动态行格式；MySQL5.7、8.0默认行格式。

* Dynamic、Compressed行格式和Compact行格式挺像，只不过在处理行溢出数据时有分歧：

  * Compressed和Dynamic两种记录格式对于存放的数据采用的是完全的行溢出的方式。

    ![image-20220703234651289](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703234651289.png)

  * Compact和Redundant两种格式会在记录的真实数据处存储一部分数据（存放768个前缀字节）。

  * Compressed行记录格式的另一个功能就是，存储在其中的行数据会以zlib的算法进行压缩，因此对于BLOB、TEXT、VARCHAR这类大长度类型的数据能够进行非常有效的存储。

### Redundant

* Redundant是MySQL 5.0版本之前InnoDB的行记录存储方式，MySQL 5.0支持Redundant是为了兼容之前版本的页格式。

* 不同于Compact行记录格式，Redundant行格式的首部是一个字段长度偏移列表，同样是按照列的顺序逆序放置的。

  ![image-20220703234755266](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/image-20220703234755266.png)

* Redundant行格式会把该条记录中所有列（包括隐藏列）的长度信息都按照逆序存储到字段长度偏移列表。

* 计算列值长度的方式不像Compact行格式那么直观，它是采用两个相邻数值的差值来计算各个列值的长度，因为存储的是字段长度的偏移量。

* 不同于Compact行格式，Redundant行格式中的记录头信息固定占用6个字节（48位）。

  ![图像](https://revenge-img.oss-cn-guangzhou.aliyuncs.com/img/%E5%9B%BE%E5%83%8F.png)

## 表空间

* 表空间可以看做是InnoDB存储引擎逻辑结构的最高层，所有的数据都存放在表空间中。

* 表空间是一个**逻辑容器**，表空间存储的对象是段，在一个表空间中可以有一个或多个段，但是一个段只能属于一个表空间。

* 表空间从管理上可划分为：系统表空间System tablespace)、独立表空间(File-per-table tablespace)、撤销表空间(Undo Tablespace)和临时表空间(Temporary Tablespace）等。

* 独立表空间：

  * 每张表有一个独立的表空间，也就是数据和索引信息都会保存在自己的表空间中。独立的表空间(即:单表)可以在不同的数据库之间进**迁移**。
  * 空间可以回收(DROPTABLE操作可自动回收表空间;其他情况，表空间不能自己回收)。如果对于统计分析或是日志表，删除大量数据后可以通过：alter table TableName engine=innodb 回收不用的空间。对于使用独立表空间的表，不管怎么删除，表空间的碎片不会太严重的影响性能，而且还有机会处理。
  * 新建的表对应的`.ibd`文件只占用了`96K`，才6个页面大小(MySQL5.7中)，这是因为一开始表空间占用的空间很小，因为表里边都没有数据。不过别忘了这些.ibd文件是`自扩展的`，随着表中数据的增多，表空间对应的文件也逐渐增大。MySQL8.0中是 7个页面大小。原因.idb 还存储了表结构，表结构.frm被取消了。

* 系统表空间：

  * 系统表空间的结构和独立表空间基本类似，只不过由于整个MySQL进程只有一个系统表空间，在系统表空间中会额外记录一些有关整个系统信息的页面，这部分是独立表空间中没有的。

  * MySQL除了保存着我们插入的用户数据之外，还需要保存许多额外的信息。

    | 表名             | 描述                                                 |
    | ---------------- | ---------------------------------------------------- |
    | SYS_TABLES       | 整个InnoDB存储引擎中所有的表的信息                   |
    | SYS_COLUMNS      | 整个InnoDB存储引擎中所有的列的信息                   |
    | SYS_INDEXES      | 整个InnoDB存储引擎中所有的索引的信息                 |
    | SYS_FIELDS       | 整个InnoDB存储引擎中所有的索引对应的列的信息         |
    | SYS_FOREIGN      | 整个InnoDB存储引擎中所有的外键的信息                 |
    | SYS_FOREIGN_COLS | 整个InnoDB存储引擎中所有的外键对应列的信息           |
    | SYS_TABLESPACES  | 整个InnoDB存储引擎中所有的表空间信息                 |
    | SYS_DATAFILES    | 整个InnoDB存储引擎中所有的表空间对应文件系统的文件路 |
    | SYS_VIRTUAL      | 整个InnoDB存储引擎中所有的虚拟生成列的信息           |

    > 这些表都存储在系统表空间中。
    >
    > 注意：用户是无法直接访问这些内部系统表的，不过考虑到查看这些表的内容可能有助于大家分析问题，所以在系统数据库**information_schema**中提供了一些以**innodb_sys**开头的表。
    >
    > ```mysql
    > USE information_schema;
    > SHOW TABLES LIKE 'innodb_sys%';
    > ```

## 扩展

* 数据页加载的三种方式：内存读取、随机读取、顺序读取。

