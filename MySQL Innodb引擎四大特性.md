---
title: MySQL Innodb引擎四大特性
date: 2023-05-28
updated : 2023-05-28
categories: 
- MySQL
tags: 
- MySQL
description: 这是一篇关于MySQL Innodb引擎四大特性的Blog。
---

## 插入缓冲

### 什么是insert buffer

insert buffer是一种特殊的数据结构（B+ tree）并不是缓存的一部分，而是物理页，当受影响的索引页不在buffer pool时缓存 secondary index pages的变化，当buffer page读入buffer pool时，进行合并操作，这些操作可以是 INSERT, UPDATE, or DELETE operations (DML)

最开始的时候只能是insert操作，所以叫做insert buffer，现在已经改叫做change buffer了

insert buffer 只适用于 non-unique secondary indexes 也就是说只能用在非唯一的索引上，原因如下

1. primary key 是按照递增的顺序进行插入的，异常插入聚族索引一般也顺序的，非随机IO
2. 写唯一索引要检查记录是不是存在，所以在修改唯一索引之前,必须把修改的记录相关的索引页读出来才知道是不是唯一、这样Insert buffer就没意义了，要读出来(随机IO)

所以只对非唯一索引有效

### insert buffer的原理

对于为非唯一索引，辅助索引的修改操作并非实时更新索引的叶子页，而是把若干对同一页面的更新缓存起来做，合并为一次性更新操 作，减少IO，转随机IO为顺序IO,这样可以避免随机IO带来性能损耗，提高数据库的写性能

具体流程

先判断要更新的这一页在不在缓冲池中

1. 若在，则直接插入；
2. 若不在，则将index page 存入Insert Buffer，按照Master Thread的调度规则来合并非唯一索引和索引页中的叶子结点

### insert buffer的内部实现

1. insert buffer的数据结构是一棵B+树，在MySQL4.1之前的版本中每张表都有一棵insert buffer B+树MySQL4.1之后，全局只有一棵insert buffer B+树，负责对所有的表的辅助索引进行 insert buffer。这棵B+树存放在共享表空间中，默认也就是ibdata1中。因此，试图通过独立表空间ibd文件恢复表中数据时，往往会导致check table 失败。这是因为表的辅助索引中的数据可能还在insert buffer中，也就是共享表空间中。所以通过idb文件进行恢复后，还需要进行repair table 操作来重建表上所有的辅助索引

2. insert buffer的非叶子节点存放的是查询的search key（键值）

   space表示待插入记录所在的表空间id，InnoDB中，每个表有一个唯一的space id，可以通过space id查询得知是哪张表；

   marker是用来兼容老版本的insert buffer；

   offset表示页所在的偏移量。

3. 当一个辅助索引需要插入到页（space， offset）时，如果这个页不在缓冲池中，那么InnoDB首先根据上述规则构造一个search key，接下来查询insert buffer这棵B+树，然后再将这条记录插入到insert buffer B+树的叶子节点中

4. 对于插入到insert buffer B+树叶子节点的记录，需要根据如下规则进行构造：
   space | marker | offset | metadata | secondary index record
   启用insert buffer索引后，辅助索引页（space、page_no）中的记录可能被插入到insert buffer B+树中，所以为了保证每次merge insert buffer页必须成功，还需要有一个特殊的页来标记每个辅助索引页（space、page_no）的可用空间，这个页的类型为insert buffer bitmap。

### insert buffer的缺点

1. 内存开销：Insert Buffer需要占用一定的内存空间来缓存待插入的数据，这会增加系统的内存开销。如果插入的数据量过大，超过了Insert Buffer的容量，那么额外的插入操作可能会直接写入磁盘，导致性能下降。
2. 数据一致性：由于Insert Buffer是一个缓冲区，数据首先被写入缓冲区，然后再异步地写入磁盘。在系统异常崩溃或断电的情况下，可能会出现数据丢失或不一致的情况。因此，在使用Insert Buffer时，需要权衡性能和数据一致性之间的折衷。
3. 额外的磁盘I/O：Insert Buffer虽然可以提高插入性能，但在后台写入磁盘时，会引入额外的磁盘I/O操作。如果磁盘I/O成为系统的瓶颈，那么Insert Buffer可能会对系统的整体性能产生负面影响。
4. 更新操作的开销：当插入的数据行与已存在的行冲突时，InnoDB引擎需要检查Insert Buffer中的数据是否需要更新已有的行。这会引入额外的开销，尤其是在有大量冲突的情况下。

### 查看insert buffer

要查看 MySQL 的 Insert Buffer 相关信息，可以使用以下方法：

1. 使用 `SHOW ENGINE INNODB STATUS` 命令，在返回的结果中查找 `BUFFER POOL AND MEMORY` 部分。在这个部分中，会显示 Insert Buffer 相关的信息，包括 Insert Buffer 的状态、大小、使用情况等。

2. 使用 `SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';` 命令，该命令会返回与 InnoDB 缓冲池相关的全局状态信息。你可以在结果中查找类似 `Innodb_buffer_pool_pages_misc`、`Innodb_buffer_pool_pages_flushed`、`Innodb_buffer_pool_pages_free` 等参数，这些参数中的值与 Insert Buffer 相关。

3. 通过查询 `INFORMATION_SCHEMA` 数据库中的相关表来获取 Insert Buffer 的信息。你可以执行以下 SQL 查询语句：

   ```sql
   SELECT * FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE_LRU;
   SELECT * FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE;
   SELECT * FROM INFORMATION_SCHEMA.INNODB_BUFFER_POOL_STATS;
   ```

   这些查询将返回 Insert Buffer 相关的统计信息、缓冲池页面的详细信息等。

请注意，具体的命令和查询语句可能会因 MySQL 版本和配置而有所不同。因此，建议根据自己的环境和需求选择适合的方法来查看 Insert Buffer 相关的信息。

## 二次写

### 脏页刷盘风险

MySQL的脏页刷盘风险指的是在数据库中存在脏页（Dirty Page）时，系统发生异常或意外宕机导致这些脏页没有及时刷写到磁盘上，从而导致数据丢失或一致性问题。

脏页是指已经被修改但尚未写入磁盘的数据库页面。在MySQL中，为了提高写入性能，数据的修改通常是在内存中进行，然后由后台线程定期将脏页刷写到磁盘上。但是，如果系统在刷写脏页之前发生异常，比如服务器崩溃、断电或人为操作错误等，那么这些脏页就无法及时刷写到磁盘上，从而导致数据的丢失或不一致。

脏页刷盘风险的存在可能会导致以下问题：

1. 数据丢失：如果系统发生异常或宕机时，脏页中的数据尚未被写入磁盘，那么这些修改将会丢失，导致数据的不完整性。

2. 数据不一致：如果系统在写入脏页之前发生异常，部分修改可能已经写入到磁盘，而另一部分修改还在内存中，这会导致数据的不一致性。

为了减少脏页刷盘风险，可以采取以下措施：

1. 合理配置数据库的持久化策略，如适当调整脏页刷新频率、写日志的方式等。

2. 使用合适的硬件设备和配置，如使用可靠的电源、磁盘阵列、RAID等技术提高数据的持久性和可靠性。

3. 定期备份数据库以及进行恢复测试，确保在发生异常情况时能够及时恢复数据。

4. 注意监控系统的运行状态，及时发现和处理异常情况，避免脏页刷盘风险的发生。

总之，脏页刷盘风险是数据库系统中需要关注的一个问题，通过合理的配置和监控手段可以减少这种风险，并确保数据的持久性和一致性。

### double write两次写

提高innodb的可靠性，用来解决部分写失败(partial page write页断裂)。

一个数据页的大小是16K，假设在把内存中的脏页写到数据库的时候，写了2K突然掉电，也就是说前2K数据是新的，后14K是旧的，那么磁盘数据库这个数据页就是不完整的，是一个坏掉的数据页。redo只能加上旧、校检完整的数据页恢复一个脏块，不能修复坏掉的数据页，所以这个数据就丢失了，可能会造成数据不一致，所以需要double write。

doublewrite由两部分组成，一部分为内存中的doublewrite buffer，其大小为2MB，另一部分是磁盘上共享表空间(ibdata x)中连续的128个页，即2个区(extent)，大小也是2M。

1. 当一系列机制触发数据缓冲池中的脏页刷新时，并不直接写入磁盘数据文件中，而是先拷贝至内存中的doublewrite buffer中；
2. 接着从两次写缓冲区分两次写入磁盘共享表空间中(连续存储，顺序写，性能很高)，每次写1MB；
3. 待第二步完成后，再将doublewrite buffer中的脏页数据写入实际的各个表空间文件(离散写)；(脏页数据固化后，即进行标记对应doublewrite数据可覆盖)
4. doublewrite的崩溃恢复
   如果操作系统在将页写入磁盘的过程中发生崩溃，在恢复过程中，innodb存储引擎可以从共享表空间的doublewrite中找到该页的一个最近的副本，将其复制到表空间文件，再应用redo log，就完成了恢复过程。因为有副本所以也不担心表空间中数据页是否损坏。

redolog写入的单位就是512字节，也就是磁盘IO的最小单位，所以无所谓数据损坏。

### double write的副作用

1. double write带来的写负载

   1. double write是一个buffer, 但其实它是开在物理文件上的一个buffer, 其实也就是file, 所以它会导致系统有更多的fsync操作, 而硬盘的fsync性能是很慢的, 所以它会降低mysql的整体性能。
   2. 但是，doublewrite buffer写入磁盘共享表空间这个过程是连续存储，是顺序写，性能非常高，(约占写的%10)，牺牲一点写性能来保证数据页的完整还是很有必要的。

2. 监控double write工作负载开启doublewrite后，每次脏页刷新必须要先写doublewrite，而doublewrite存在于磁盘上的是两个连续的区，每个区由连续的页组成，一般情况下一个区最多有64个页，所以一次IO写入应该可以最多写64个页。

   而根据以上系统Innodb_dblwr_pages_written与Innodb_dblwr_writes的比例来看，大概在3左右，远远还没到64(如果约等于64，那么说明系统的写压力非常大，有大量的脏页要往磁盘上写)，所以从这个角度也可以看出，系统写入压力并不高。

3. 关闭double write适合的场景

   1. 海量DML
   2. 不惧怕数据损坏和丢失
   3. 系统写负载成为主要负载


   作为InnoDB的一个关键特性，doublewrite功能默认是开启的，但是在上述特殊的一些场景也可以视情况关闭，来提高数据库写性能。静态参数，配置文件修改，重启数据库。

4. 为什么没有把double write里面的数据写到data page里面

   1. double write里面的数据是连续的，如果直接写到data page里面，而data page的页又是离散的，写入会很慢。
   2. double write里面的数据没有办法被及时的覆盖掉，导致double write的压力很大；短时间内可能会出现double write溢出的情况。

## 自适应哈希索引

### 索引的资源消耗分析

索引的资源消耗点

- 树的高度，顺序访问索引的数据页，索引就是在列上建立的，数据量非常小，在内存中；

- 数据之间跳着访问

  1. 索引往表上跳，可能需要访问表的数据页很多；

  2. 通过索引访问表，主键列和索引的有序度出现严重的不一致时，可能就会产生大量物理读；

资源消耗最厉害：通过索引访问多行，需要从表中取多行数据，如果无序的话，来回跳着找，跳着访问，物理读会很严重。

### 自适应哈希索引原理

1. 查询模式分析：MySQL会监控每个索引的使用情况，统计索引的访问频率和效率。根据查询模式的统计信息，MySQL会判断哪些索引适合创建哈希索引。
2. 哈希索引的创建：当某个索引被频繁访问且效率较低时，MySQL会自动创建一个哈希索引。哈希索引是一种内存索引，将索引键映射到哈希表中的桶（Bucket）。每个桶包含指向实际数据行的指针。
3. 动态调整：MySQL会动态调整哈希索引的大小，以适应数据量的变化和查询模式的变化。当索引被频繁访问时，MySQL会增加哈希表的大小，以提供更好的查询性能。当索引使用较少时，MySQL会减小哈希表的大小，以节省内存资源。

自适应哈希索引的优点是能够根据实际情况自动选择和调整索引，提高查询的性能和效率。它可以减少磁盘IO，因为哈希索引存储在内存中，提供了更快的数据访问速度。此外，自适应哈希索引对于查询模式的变化也具有很好的适应性，可以动态调整索引以满足不同查询的需求。

需要注意的是，自适应哈希索引适用于一些特定的查询模式和数据分布情况。在某些场景下，使用其他类型的索引（如B+树索引）可能更适合。因此，在实际应用中，需要根据具体情况进行索引选择和性能测试，以获得最佳的查询性能。

经常访问的二级索引数据会自动被生成到hash索引里面去(最近连续被访问三次的数据)，自适应哈希索引通过缓冲池的B+树构造而来，因此建立的速度很快。

特点：

1. 无序，没有树高
2. 降低对二级索引树的频繁访问资源
   索引树高<=4，访问索引：访问树、根节点、叶子节点
3. 自适应

缺陷：

1. hash自适应索引会占用innodb buffer pool；
2. 自适应hash索引只适合搜索等值的查询，如select * from table where index_col='xxx'，而对于其他查找类型，如范围查找，是不能使用的；
3. 极端情况下，自适应hash索引才有比较大的意义，可以降低逻辑读。

## 预读

InnoDB使用两种预读算法来提高I/O性能：线性预读(linear read-ahead)和随机预读(randomread-ahead)

为了区分这两种预读的方式，可以把线性预读放到以extent为单位，而随机预读放到以extent中的page为单位。线性预读着眼于将下一个extent提前读取到buffer pool中，而随机预读着眼于将当前extent中的剩余的page提前读取到buffer pool中。

### 线性预读

线性预读有一个很重要的变量控制是否将下一个extent预读到buffer pool中，通过使用配置参数innodb_read_ahead_threshold，可以控制Innodb执行预读操作的时间。如果一个extent中的被顺序读取的page超过或者等于该参数变量时，Innodb将会异步的将下一个extent读取到buffer pool中，innodb_read_ahead_threshold可以设置为0-64的任何值，默认值为56，值越高，访问模式检查越严格。

例如，如果将值设置为48，则InnoDB只有在顺序访问当前extent中的48个pages时才触发线性预读请求，将下一个extent读到内存中。如果值为8，InnoDB触发异步预读，即使程序段中只有8页被顺序访问。你可以在MySQL配置文件中设置此参数的值，或者使用SET GLOBAL需要该SUPER权限的命令动态更改该参数。

在没有该变量之前，当访问到extent的最后一个page的时候，Innodb会决定是否将下一个extent放入到buffer pool中。

### 随机预读

随机预读则是表示当同一个extent中的一些page在buffer pool中发现时，Innodb会将该extent中的剩余page一并读到buffer pool中，由于随机预读方式给Innodb code带来了一些不必要的复杂性，同时在性能也存在不稳定性，在5.5中已经将这种预读方式废弃。要启用此功能，请将配置变量设置innodb_random_read_ahead为ON。