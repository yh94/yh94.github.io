---
title: mysql-binlog
date: 2022-1-10
tags:
 - java
 - mysql
categories:
 - java
 - mysql
---
[MySQL如何开启binlog？binlog三种模式的分析](https://www.jianshu.com/p/8e7e288c41b1)

> 前提，创建表t，并插入数据，语句如下：

```
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `t_modified`(`t_modified`)
) ENGINE=InnoDB;

insert into t values(1,1,'2018-11-13');
insert into t values(2,2,'2018-11-12');
insert into t values(3,3,'2018-11-11');
insert into t values(4,4,'2018-11-10');
insert into t values(5,5,'2018-11-09');

```

#### 判断MySQL是否已经开启binlog?

登录mysql，执行：SHOW VARIABLES LIKE 'log\_bin';

*   OFF：关闭
*   ON：开启

#### 如何开启binlog日志？

找到my.cnf ：

```
select @@basedir;

```

如果还是找不到文件的位置，可以执行下面命令，可以得到mysql的配置文件的默认加载顺序

```
mysql --help | grep 'Default options' -A 1

```

执行结果：

```
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf

```

可以看到mysql优先加载`/etc/my.cnf`中的配置。

所以需要在`/etc/my.cnf`中mysqld节添加开启binlog的配置，如下有两种方式：

```
#第一种方式:
#开启binlog日志
log_bin=ON
#binlog日志的基本文件名
log_bin_basename=/var/lib/mysql/mysql-bin
#binlog文件的索引文件，管理所有binlog文件
log_bin_index=/var/lib/mysql/mysql-bin.index
#配置serverid
server-id=1

#第二种方式:
#此一行等同于上面log_bin三行
log-bin=/var/lib/mysql/mysql-bin
#配置serverid
server-id=1

```

修改完配置后，重启mysql。执行`SHOW VARIABLES LIKE 'log_bin';` Value 值为 ON即可。

### binlog的配置简介

MySQL配置文件my.cnf文件中的mysqld节的配置：

```
[mysqld]
#设置日志三种格式：STATEMENT、ROW、MIXED 。
binlog_format = mixed
#设置日志路径，注意路经需要mysql用户有权限写
log-bin = /data/mysql/logs/mysql-bin.log
#设置binlog清理时间
expire_logs_days = 7
#binlog每个日志文件大小
max_binlog_size = 100m
#binlog缓存大小
binlog_cache_size = 4m
#最大binlog缓存大小
max_binlog_cache_size = 512m

```

重启MySQL生效，如果不方便重启服务，也可以直接修改对应的变量即可。

### binlog的三种模式比较

binlog的格式也有三种：`STATEMENT`、`ROW`、`MIXED`。

*   `STATMENT`模式：基于SQL语句的复制(statement-based replication, SBR)，每一条会修改数据的sql语句会记录到binlog中。  
    优点：不需要记录每一条SQL语句与每行的数据变化，这样子binlog的日志也会比较少，减少了磁盘IO，提高性能。  
    缺点：在某些情况下会导致master-slave中的数据不一致(比如：`delete from t where a>=4 and t_modified<='2018-11-10' limit 1;`在主库执行这个语句的时候，如果使用的是a索引，会删除`(4,4,'2018-11-10')`这条记录，如果使用的是t\_modified的索引则会删除`insert into t values(5,5,'2018-11-09');`所以在执行这条sql语句的时候提示：  
    `Unsafe statement written to the binary log using statement format since BINLOG_FORMAT = STATEMENT. The statement is unsafe because it uses a LIMIT clause. This is unsafe because the set of rows included cannot be predicted.`  
    由于 statement 格式下，记录到 binlog 里的是语句原文，因此可能会出现这样一种情况：在主库执行这条 SQL 语句的时候，用的是索引 a；而在备库执行这条 SQL 语句的时候，却使用了索引 t\_modified。因此，MySQL 认为这样写是有风险的。  
    `sleep()`函数， `last_insert_id()`，以及`user-defined functions(udf)`等也会出现问题)；
*   `ROW`基于行的复制(row-based replication, RBR)格式：不记录每一条SQL语句的上下文信息，仅需记录哪条数据被修改了，修改成了什么样子了。  
    优点：不会出现某些特定情况下的存储过程、或function、或trigger的调用和触发无法被正确复制的问题。  
    缺点：会产生大量的日志，尤其是alter table的时候会让日志暴涨。
*   `MIXED`混合模式复制(mixed-based replication, MBR)：以上两种模式的混合使用，一般的复制使用STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存binlog，MySQL会根据执行的SQL语句选择日志保存方式。

### 列举几个关于binlog常用的命令

```
#查看日志开启状态 
show variables like 'log_%';
#查看所有binlog日志列表
show master logs;
#查看最新一个binlog日志的编号名称，及其最后一个操作事件结束点 
show master status;
#刷新log日志，立刻产生一个新编号的binlog日志文件，跟重启一个效果 
flush logs;
#清空所有binlog日志 
reset master;

```

### binlog中到底记录了什么内容？如何分析binlog日志？

开启binlog日志之后，我们在登录mysql，在控制台执行一条sql语句：`delete from t where a>=4 and t_modified<='2018-11-10' limit 1;`  
查看当前的binlog日志文件`show master status;`;  
笔者得到的是`mysql-bin.000005`，根据binlog日志文件的配置，我们可以去存储binlog的日志文件下面看看`cd /var/lib/mysql/`,可以看到文件夹下面有`mysql-bin.000005`文件。  
执行：cat mysql-bin.000005，结果如下：

```
[root@mysql]# cat mysql-bin.000005
n
 MZ]w{5.7.27-log
_               MZ]8


**4Z]#

```

可以看到，里面的内容根本没法阅读。

因为binlog日志文件：mysql-bin.000005是二进制文件，没法用vi等打开，这时就需要mysql的自带的mysqlbinlog工具进行解码，执行：`mysqlbinlog mysql-bin.000005`可以将二进制文件转为可阅读的sql语句。

### 分析对比binlog的`ROW`模式和`STATEMENT`模式下的日志：

`ROW`模式：

```
    /*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
    /*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
    DELIMITER /*!*/;
    # at 4
    #190819 15:35:46 server id 1  end_log_pos 123 CRC32 0x2bfa58f0  Start: binlog v 4, server v 5.7.27-log created 190819 15:35:46 at startup
    # Warning: this binlog is either in use or was not closed properly.
    ROLLBACK/*!*/;
    BINLOG '
    UlFaXQ8BAAAAdwAAAHsAAAABAAQANS43LjI3LWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
    AAAAAAAAAAAAAAAAAABSUVpdEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
    AfBY+is=
    '/*!*/;
    # at 123
    #190819 15:35:46 server id 1  end_log_pos 154 CRC32 0xe4e91c8f  Previous-GTIDs
    # [empty]
    # at 154
    #190819 15:36:19 server id 1  end_log_pos 219 CRC32 0xcc3dc023  Anonymous_GTID  last_committed=0        sequence_number=1       rbr_only=yes
    /*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
    SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
    # at 219
    #190819 15:36:19 server id 1  end_log_pos 305 CRC32 0xd39f98cf  Query   thread_id=2     exec_time=0     error_code=0
    SET TIMESTAMP=1566200179/*!*/;
    SET @@session.pseudo_thread_id=2/*!*/;
    SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
    SET @@session.sql_mode=1436549152/*!*/;
    SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
    /*!\C utf8 *//*!*/;
    SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
    SET @@session.time_zone='SYSTEM'/*!*/;
    SET @@session.lc_time_names=0/*!*/;
    SET @@session.collation_database=DEFAULT/*!*/;
    BEGIN
    /*!*/;
    # at 305
    #190819 15:36:19 server id 1  end_log_pos 358 CRC32 0x56a1e39d  Table_map: `mysql_test`.`t` mapped to number 108
    # at 358
    #190819 15:36:19 server id 1  end_log_pos 406 CRC32 0xc91237b0  Delete_rows: table id 108 flags: STMT_END_F
    
    BINLOG '
    c1FaXRMBAAAANQAAAGYBAAAAAGwAAAAAAAEACm15c3FsX3Rlc3QAAXQAAwMDEQEAAp3joVY=
    c1FaXSABAAAAMAAAAJYBAAAAAGwAAAAAAAEAAgAD//gEAAAABAAAAFvlrwCwNxLJ
    '/*!*/;
    ### DELETE FROM `mysql_test`.`t`
    ### WHERE
    ###   @1=4 /* INT meta=0 nullable=0 is_null=0 */
    ###   @2=4 /* INT meta=0 nullable=1 is_null=0 */
    ###   @3=1541779200 /* TIMESTAMP(0) meta=0 nullable=0 is_null=0 */
    # at 406
    #190819 15:36:19 server id 1  end_log_pos 437 CRC32 0x7898ebf6  Xid = 13
    COMMIT/*!*/;
    SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
    DELIMITER ;
    # End of log file
    /*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
    /*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;

```

`STATEMENT`模式：

```
    /*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
    /*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
    DELIMITER /*!*/;
    # at 4
    #190819 16:19:48 server id 1  end_log_pos 123 CRC32 0xec3e6426  Start: binlog v 4, server v 5.7.27-log created 190819 16:19:48 at startup
    # Warning: this binlog is either in use or was not closed properly.
    ROLLBACK/*!*/;
    BINLOG '
    pFtaXQ8BAAAAdwAAAHsAAAABAAQANS43LjI3LWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
    AAAAAAAAAAAAAAAAAACkW1pdEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
    ASZkPuw=
    '/*!*/;
    # at 123
    #190819 16:19:48 server id 1  end_log_pos 154 CRC32 0xc6db211a  Previous-GTIDs
    # [empty]
    # at 154
    #190819 16:20:01 server id 1  end_log_pos 219 CRC32 0x1960cba5  Anonymous_GTID  last_committed=0        sequence_number=1       rbr_only=no
    SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
    # at 219
    #190819 16:20:01 server id 1  end_log_pos 318 CRC32 0x11eecb38  Query   thread_id=2     exec_time=0     error_code=0
    SET TIMESTAMP=1566202801/*!*/;
    SET @@session.pseudo_thread_id=2/*!*/;
    SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
    SET @@session.sql_mode=1436549152/*!*/;
    SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
    /*!\C utf8 *//*!*/;
    SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
    SET @@session.time_zone='SYSTEM'/*!*/;
    SET @@session.lc_time_names=0/*!*/;
    SET @@session.collation_database=DEFAULT/*!*/;
    BEGIN
    /*!*/;
    # at 318
    #190819 16:20:01 server id 1  end_log_pos 473 CRC32 0x390d1f96  Query   thread_id=2     exec_time=0     error_code=0
    use `mysql_test`/*!*/;
    SET TIMESTAMP=1566202801/*!*/;
    delete from t where a>=4 and t_modified<='2018-11-10' limit 1
    /*!*/;
    # at 473
    #190819 16:20:01 server id 1  end_log_pos 504 CRC32 0x2f35c773  Xid = 13
    COMMIT/*!*/;
    SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
    DELIMITER ;
    # End of log file
    /*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
    /*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;

```

可以看到`ROW`模式下，binlog日志中的begin和commit之间并没有sql语句。  
在`STATEMENT`模式下，binlog日志中的begin和commit之间是一条sql语句。

