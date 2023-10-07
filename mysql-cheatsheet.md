- 启动查看配置文件目录

  ```shell
  mysql --help | grep my.cnf
  # output
   order of preference, my.cnf, $MYSQL_TCP_PORT,
  /etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf
  ```

- 查看数据目录

  ```mysql
  show variables like 'datadir'
  ```

- show ENGINES

- 测试数据：https://dev.mysql.com/doc/index-other.html

- show variables like 'innodb_read_io_threads'

- show variables like 'innodb_write_io_threads'

- purge thread（show variables like 'innodb_purge_threads'）：回收undo页，innodb_purge_batch_size每次回收undo页的个数

- buffer pool（show variables like 'innodb_buffer_pool_size'）：

  - 配置实例个数：show variables like 'innodb_buffer_pool_instances'，每个页根据hash值分配到不同的缓存实例中，减小内存竞争
    - select POOL_ID, POOL_SIZE,FREE_BUFFERS, DATABASE_PAGES FROM information_schema.INNODB_BUFFER_POOL_STATS

- checkpoint：将buffer pool中的脏页刷新回磁盘

- 内存数据

  ![](./images/innodb_memory_data.png)

- LRU：show variables like 'innodb_old_blocks_pct'（默认：37（百分比），大约5/8之后部分）

- buffer pool状态：

  ```mysql
  show engine innodb status
  -- （不是当前状态，Per second averages calculated from the last 2 seconds）
  -- buffer pool size: 缓冲池页的个数，乘以16K，就是缓冲池的总大小
  -- free buffers:free列表页的个数
  -- database pages：LRU列表页的数量
  -- buffer pool hit rate：缓冲池命中率，一般不低于95%
  
  -- 相同效果：
  SELECT POOL_ID,HIT_RATE,PAGES_MADE_YOUNG,PAGES_NOT_MADE_YOUNG FROM information_schema.INNODB_BUFFER_POOL_STATS
  
  -- 查看缓冲池LRU列表某个SPACE的页类型
  SELECT TABLE_NAME,SPACE,PAGE_NUMBER,PAGE_TYPE FROM information_schema.INNODB_BUFFER_PAGE_LRU WHERE SPACE=4294967294
  ```

- flush list：脏页（Modified db pages）既存在于LRU列表，也存在于flush list。LRU用于管理页的可用性，flush list管理将脏页刷新回磁盘。

- innodb_log_buffer_size：show variables like 'innodb_log_buffer_size'，单位字节，不需要设置得很大，因为每一秒都会将buffer中的日志刷新回磁盘。

- Checkpoint：将缓冲池中的脏页刷新回磁盘。DML，修改或删除操作先发生在缓冲池中，后续将这些脏页刷新回磁盘。通过write ahead log策略，当事务提交时，先写redo log，再修改页，防止发生宕机引起数据丢失问题。

  - 解决的问题
    - 缩短数据库的恢复时间（循环写redo log，将已经提交，且脏页已经同步回磁盘的日志部分进行回收，防止redo log文件无限扩张，缩短数据库宕机恢复时间，因为只需要从redo log文件中恢复checkpoint之后的数据）
    - 缓冲池不够用时，将脏页刷新回磁盘（根据LRU淘汰页，如果是脏页，就强制执行checkpoint，将脏页刷新回磁盘）
    - 重做日志不可用时，刷新脏页回磁盘（redo log文件不够写，需要将脏页同步回磁盘，回收日志文件空间）
  - LSN（log sequence number）：页、redo log、checkpoint都有LSN。
  - fuzzy checkpoint发生时机
    - master thread checkpoint
    - flush_lru_list checkpoint：innodb_lru_scan_depth，控制LRU列表可用页的数量
    - async/sync flush checkpoint：redo log文件不够用，强制刷新脏页回磁盘，回收对应的日志文件空间。
    - dirty page too much checkpoint：innodb_max_dirty_pages_pct，控制脏页比例，超过则执行checkpoint。
  - master thread:
    - show variables like 'innodb_io_capacity'（默认值：200）：
      - merge insert buffer页数量为此值5%
      - 刷新脏页数量为此值
    - 执行full purge时，每次回收的undo页数量：innodb_purge_batch_size（默认300）
  
- change buffer（insert\delete\update）：适用于非唯一的二级辅助索引，非聚集索引是离散存储的，进行插入时需要离散的访问非聚集索引数据页；和buffer pool中的数据一样，也具有磁盘数据。先判断数据是否在buffer pool中，如果不在，就先存入change buffer中，后续再将多个插入合并到一个操作中。

  - show engine innodb status

    ```
    Ibuf: size 1, free list len 0, seg size 2, 0 merges
    merged operations:
     insert 0, delete mark 0, delete 0
    discarded operations:
     insert 0, delete mark 0, delete 0
     # seg size代表数据页个数。insert代表insert buffer，delete mark代表 delete buffer，delete代表purge buffer，discarded operations代表change buffer发生merge时，表已经被删了，此时无需合并到buffer pool中。
    ```

    

  - 存在的问题：

    - 如果进行了大量的插入操作，insert buffer中的数据没有merge到buffer pool中，会导致重启时间过长
    - 暂用过多的buffer pool内存。

  - innodb_change_buffering

  - innodb_change_buffer_max_size（默认为25，占用四分之一，最大有效值50）

  - 合并change buffer到buffer pool

    - 辅助索引页被读取到buffer pool。
    - insert buffer bitmap页追踪到辅助索引页没有可用空间
    - master  thread

- double write：

  > 保证数据页的可靠性。如果buffer pool中的脏页在flush到磁盘的过程中，数据库或操作系统宕机，会导致原始磁盘上的数据不完整，此时使用redo log是无意义的，因为原始数据已经被破坏了。

  - 解决方案：通过double write保存一个buffer pool中脏页在磁盘和内存上的一个的副本。刷新脏页时，先不直接写磁盘，而是将脏页数据复制到double write buffer中，内存区域大小为2MB，再分两次顺序写入double write的磁盘共享表空间位置，通过一次调用fsync。double write写好后，再将double write buffer页中的数据写入各个表空间

​		![](./images/double_write.png)

```mysql
show global status like 'innodb_dblwr%'
Innodb_dblwr_pages_written # 写入数据页个数
Innodb_dblwr_writes	# 实际写入次数
Innodb_dblwr_pages_written/Innodb_dblwr_writes如果远小于61:1，说明系统写入压力不大
```

