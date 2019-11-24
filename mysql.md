





```shell
 SET @@GLOBAL.GTID_PURGED='ceabbacf-0c77-11ea-b49f-2016d8c96b46:1-4';

# 保证server id唯一
SET @@GLOBAL.GTID_PURGED='ceabbacf-0c77-11ea-b49f-2016d8c96b46:1-4';
# 在单机多实例部署的时候，在data/auto.cnf文件中的server uuid值在同一个机器中是相同的，如果执行SET @@GLOBAL.GTID_PURGED='ceabbacf-0c77-11ea-b49f-2016d8c96b46:1-4';这个命令时，会报错如下：
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ' SET @@GLOBAL.GTID_PURGED='ceabbacf-0c77-11ea-b49f-2016d8c96b46:1-4'' at line 1
即gtid复制不能让uuid相同的机器之间建立复制关系，因此，需要手动修改该文件，使其整个集群是惟一的。


CHANGE MASTER TO MASTER_HOST='127.0.0.1',          
MASTER_PORT=3306,                                     
MASTER_USER='tmp',                             
MASTER_PASSWORD='tmp',   
MASTER_AUTO_POSITION=1;


# server-id也不能相同，如果相同，其他步骤都不会错，但是在startslave后的io线程会出错


###################################################################################
             若备份的时候set_gtid_purged=ON，则可以按照如下两种方式处理
###################################################################################
一、第一种方式处理
# 去掉dump文件中的set @@GLOBAL.GTID_PURGED语句
mysql -h127.0.0.1 -P3308 -utmp -ptmp

source dump.sql
stop slave;
reset master

SET @@GLOBAL.GTID_PURGED='108cc4a4-0d40-11ea-9598-2016d8c96b66:1-5,c42216ad-0d37-11ea-b163-2016d8c96b56:1-16,ceabbacf-0c77-11ea-b49f-2016d8c96b46:1-1662594';

CHANGE MASTER TO MASTER_HOST='127.0.0.1',          
MASTER_PORT=3306,                                     
MASTER_USER='tmp',                             
MASTER_PASSWORD='tmp',   
MASTER_AUTO_POSITION=1;

set global read_only=1;

一、第二种方式处理
# 不去掉dump文件中的set @@GLOBAL.GTID_PURGED语句
mysql -h127.0.0.1 -P3308 -utmp -ptmp
stop slave;
reset master;
source dump.sql;
CHANGE MASTER TO MASTER_HOST='127.0.0.1',          
MASTER_PORT=3306,                                     
MASTER_USER='tmp',                             
MASTER_PASSWORD='tmp',   
MASTER_AUTO_POSITION=1;
set global read_only=1;

###################################################################################
            在备份的时候我把set_gtid_purged=OFF 从库这时候就找不到gtid去同步数据了
###################################################################################
第一种方式：
按照传统方式搭建主从复制，直接找到binlog的恢复点，按照偏移量的方式去指定主库
第二种方式：
mysql -h127.0.0.1 -P3308 -utmp -ptmp
stop slave;
reset master;
source dump.sql;
set global read_only=1;

###################################################################################
                                 set_gtid_purged
###################################################################################
我们备份，就是可能需要拿来进行恢复，是在master上恢复，还是slave上恢复。
如果是在master上进行恢复，那么就需要生成对应的gtid,所以需要使用set-gtid-purged=off
如果是在slave上进行恢复，那么不需要生成对应的gtid，所以需要使用set-gtid-purged=on
加了--set-gtid-purged=OFF时，在会记录binlog日志，如果不加，不记录binlog日志，所以在我们做主从用了gtid时，用mysqldump备份时就要加--set-gtid-purged=OFF，否则你在主上导入恢复了数据，主没有了binlog日志，同步则不会被同步。
```





## 二进制日志格式分析

```shell
# at 179666727
#191123 20:19:52 server id 3232266753  end_log_pos 179666788    GTID    last_committed=557619   sequence_number=557620  rbr_only=no
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662575'/*!*/;
# at 179666788
#191123 20:19:52 server id 3232266753  end_log_pos 179666957    Query   thread_id=557624        exec_time=0     error_code=0
use `google`/*!*/;
SET TIMESTAMP=1574511592/*!*/;
create table tb_person(id bigint(20) not null auto_increment,name varchar(255),primary key(id))
/*!*/;
# at 179666957
#191123 20:23:50 server id 3232266753  end_log_pos 179667018    GTID    last_committed=557620   sequence_number=557621  rbr_only=no
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662576'/*!*/;
# at 179667018
#191123 20:23:50 server id 3232266753  end_log_pos 179667185    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574511830/*!*/;
create table tb_host(id bigint(20) not null auto_increment,name varchar(255),primary key(id))
/*!*/;
# at 179667185
#191123 20:43:26 server id 3232266753  end_log_pos 179667246    GTID    last_committed=557621   sequence_number=557622  rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662577'/*!*/;
# at 179667246
#191123 20:43:26 server id 3232266753  end_log_pos 179667316    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574513006/*!*/;
BEGIN
/*!*/;
# at 179667316
#191123 20:43:26 server id 3232266753  end_log_pos 179667373    Rows_query
# insert into tb_host(name) values('5')
# at 179667373
#191123 20:43:26 server id 3232266753  end_log_pos 179667424    Table_map: `google`.`tb_host` mapped to number 109
# at 179667424
#191123 20:43:26 server id 3232266753  end_log_pos 179667467    Write_rows: table id 109 flags: STMT_END_F
### INSERT INTO `google`.`tb_host`
### SET
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='5' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
# at 179667467
#191123 20:43:26 server id 3232266753  end_log_pos 179667494    Xid = 2788131
COMMIT/*!*/;
# at 179667494
#191123 20:52:02 server id 3232266753  end_log_pos 179667555    GTID    last_committed=557622   sequence_number=557623  rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662578'/*!*/;
# at 179667555
#191123 20:52:02 server id 3232266753  end_log_pos 179667625    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574513522/*!*/;
BEGIN
/*!*/;
# at 179667625
#191123 20:52:02 server id 3232266753  end_log_pos 179667684    Rows_query
# update tb_host set name='10' where id=1
# at 179667684
#191123 20:52:02 server id 3232266753  end_log_pos 179667735    Table_map: `google`.`tb_host` mapped to number 109
# at 179667735
#191123 20:52:02 server id 3232266753  end_log_pos 179667792    Update_rows: table id 109 flags: STMT_END_F
### UPDATE `google`.`tb_host`
### WHERE
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='5' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
### SET
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='10' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
# at 179667792
#191123 20:52:02 server id 3232266753  end_log_pos 179667819    Xid = 2788133
COMMIT/*!*/;
# at 179667819
#191123 21:01:24 server id 3232266753  end_log_pos 179667880    GTID    last_committed=557623   sequence_number=557624  rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662579'/*!*/;
# at 179667880
#191123 21:01:24 server id 3232266753  end_log_pos 179667950    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574514084/*!*/;
BEGIN
/*!*/;
# at 179667950
#191123 21:01:24 server id 3232266753  end_log_pos 179668000    Rows_query
# delete from tb_host where id=1
# at 179668000
#191123 21:01:24 server id 3232266753  end_log_pos 179668051    Table_map: `google`.`tb_host` mapped to number 109
# at 179668051
#191123 21:01:24 server id 3232266753  end_log_pos 179668095    Delete_rows: table id 109 flags: STMT_END_F
### DELETE FROM `google`.`tb_host`
### WHERE
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='10' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
# at 179668095
#191123 21:01:24 server id 3232266753  end_log_pos 179668122    Xid = 2788134
COMMIT/*!*/;
# at 179668122
#191123 21:11:42 server id 3232266753  end_log_pos 179668183    GTID    last_committed=557624   sequence_number=557625  rbr_only=no
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662580'/*!*/;
# at 179668183
#191123 21:11:42 server id 3232266753  end_log_pos 179668303    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574514702/*!*/;
SET @@session.pseudo_thread_id=557624/*!*/;
DROP TABLE `tb_host` /* generated by server */
/*!*/;
# at 179668303
#191123 21:19:09 server id 3232266753  end_log_pos 179668364    GTID    last_committed=557625   sequence_number=557626  rbr_only=no
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662581'/*!*/;
# at 179668364
#191123 21:19:09 server id 3232266753  end_log_pos 179668493    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574515149/*!*/;
alter table tb_person add column age smallint default 0
/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
####################################################
####################################################
####################################################
MySQL的二进制日志binlog可以说是MySQL最重要的日志，它记录了所有的DDL和DML语句（除了数据查询语句select），以事件形式记录，还包含语句所执行的消耗的时间，MySQL的二进制日志是事务安全型的。

MySQL binlog记录的所有操作实际上都有对应的事件类型的，譬如STATEMENT格式中的DML操作对应的是QUERY_EVENT类型，ROW格式下的DML操作对应的是ROWS_EVENT类型。
一、对于ROW格式的binlog，所有的DML语句都是记录在ROWS_EVENT中。
ROWS_EVENT分为三种：WRITE_ROWS_EVENT，UPDATE_ROWS_EVENT，DELETE_ROWS_EVENT，分别对应insert，update和delete操作。
1、对于insert操作，WRITE_ROWS_EVENT包含了要插入的数据
2、对于update操作，UPDATE_ROWS_EVENT不仅包含了修改后的数据，还包含了修改前的值。
3、对于delete操作，仅仅需要指定删除的主键（在没有主键的情况下，会给定所有列）
二、对于QUERY_EVENT事件，是以文本形式记录DML操作的。而对于ROWS_EVENT事件，并不是文本形式，所以在通过mysqlbinlog查看基于ROW格式的binlog时，需要指定-vv --base64-output=decode-rows。

mysqlbinlog常见的选项有以下几个：
--start-datetime：从二进制日志中读取指定等于时间戳或者晚于本地计算机的时间
--stop-datetime：从二进制日志中读取指定小于时间戳或者等于本地计算机的时间 取值和上述一样
--start-position：从二进制日志中读取指定position 事件位置作为开始。
--stop-position：从二进制日志中读取指定position 事件位置作为事件截至


########################################################################################################################################
##################日志文件开头：
Previous-GTIDs：ceabbacf-0c77-11ea-b49f-2016d8c96b46:1-1662585 # 表示前一个binlog中gtid值已经执行这个位置了。

# binlog版本，mysql服务器版本，binlog文件创建时间






/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#191123 23:21:49 server id 3232266753  end_log_pos 123  Start: binlog v 4, server v 5.7.27-log created 191123 23:21:49
# Warning: this binlog is either in use or was not closed properly.
# at 123
#191123 23:21:49 server id 3232266753  end_log_pos 190  Previous-GTIDs
# ceabbacf-0c77-11ea-b49f-2016d8c96b46:1-1662585










########################################################################################################################################
################下面是共同的结尾，是二进制日志结尾的标志
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;


######################################
########stop事件：
mysql服务停止时，会在当前的binlog日志添加一个stop事件,下次重启服务后会新开一个binlog日志，故当前的binlog日志必须给以标记：
at
stop # 表示stop事件

# at 356010155
#191123  4:03:12 server id 3232266753  end_log_pos 356010174    Stop
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;

#########################################
##############Rotate事件：
当binlog文件的大小达到max_binlog_size的值或者执行flush logs命令时，binlog会发生切换，这个时候会在当前的binlog日志添加一个ROTATE_EVENT事件，用于指定下一个日志的名称和位置：
at
Rotate to mysql-bin.000012  pos: 4 #表示下一个binlog日志的文件名称和位置

# at 179670297
#191123 23:21:49 server id 3232266753  end_log_pos 179670340    Rotate to mysql-bin.000012  pos: 4
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;

########################################################################################################################################
建表语句格式：以at开头，以“/*!*/;”结尾


# at 179666957
#191123 20:23:50 server id 3232266753  end_log_pos 179667018    GTID    last_committed=557620   sequence_number=557621  rbr_only=no
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662576'/*!*/;
# at 179667018
#191123 20:23:50 server id 3232266753  end_log_pos 179667185    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574511830/*!*/;
create table tb_host(id bigint(20) not null auto_increment,name varchar(255),primary key(id))
/*!*/;




########################################################################################################################################
删除表语句格式：占用一个gtid_next
开头：at
结尾：/*!*/

# at 179668122
#191123 21:11:42 server id 3232266753  end_log_pos 179668183    GTID    last_committed=557624   sequence_number=557625  rbr_only=no
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662580'/*!*/;
# at 179668183
#191123 21:11:42 server id 3232266753  end_log_pos 179668303    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574514702/*!*/;
SET @@session.pseudo_thread_id=557624/*!*/;
DROP TABLE `tb_host` /* generated by server */
/*!*/;



########################################################################################################################################
修改表结构语句格式：占用一个gtid_next
开头：at
结尾：/*!*/

# at 179668303
#191123 21:19:09 server id 3232266753  end_log_pos 179668364    GTID    last_committed=557625   sequence_number=557626  rbr_only=no
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662581'/*!*/;
# at 179668364
#191123 21:19:09 server id 3232266753  end_log_pos 179668493    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574515149/*!*/;
alter table tb_person add column age smallint default 0
/*!*/;




########################################################################################################################################
insert语句：Write_rows类型，占用一个gtid_next
事件头以at开头，
事件体开头：
“SET TIMESTAMP=1574513006/*!*/;
BEGIN
/*!*/;” 
# 中间是事件的具体内容
事件体结尾：“COMMIT/*!*/;”


# at 179667185
#191123 20:43:26 server id 3232266753  end_log_pos 179667246    GTID    last_committed=557621   sequence_number=557622  rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662577'/*!*/;
# at 179667246
#191123 20:43:26 server id 3232266753  end_log_pos 179667316    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574513006/*!*/;
BEGIN
/*!*/;
# at 179667316
#191123 20:43:26 server id 3232266753  end_log_pos 179667373    Rows_query
# insert into tb_host(name) values('5')
# at 179667373
#191123 20:43:26 server id 3232266753  end_log_pos 179667424    Table_map: `google`.`tb_host` mapped to number 109
# at 179667424
#191123 20:43:26 server id 3232266753  end_log_pos 179667467    Write_rows: table id 109 flags: STMT_END_F
### INSERT INTO `google`.`tb_host`
### SET
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='5' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
# at 179667467
#191123 20:43:26 server id 3232266753  end_log_pos 179667494    Xid = 2788131
COMMIT/*!*/;

############################################################
测试一次插入多行：
发现：Write_rows这个内容与插入的数据量成正比。多个insert set语句

# at 179668493
#191123 22:05:11 server id 3232266753  end_log_pos 179668554    GTID    last_committed=557626   sequence_number=557627  rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662582'/*!*/;
# at 179668554
#191123 22:05:11 server id 3232266753  end_log_pos 179668624    Query   thread_id=557625        exec_time=0     error_code=0
SET TIMESTAMP=1574517911/*!*/;
BEGIN
/*!*/;
# at 179668624
#191123 22:05:11 server id 3232266753  end_log_pos 179668708    Rows_query
# insert into tb_person (name,age) values('1',1),('5',5),('10',10)
# at 179668708
#191123 22:05:11 server id 3232266753  end_log_pos 179668762    Table_map: `google`.`tb_person` mapped to number 111
# at 179668762
#191123 22:05:11 server id 3232266753  end_log_pos 179668836    Write_rows: table id 111 flags: STMT_END_F
### INSERT INTO `google`.`tb_person`
### SET
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='1' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=1 /* SHORTINT meta=0 nullable=1 is_null=0 */
### INSERT INTO `google`.`tb_person`
### SET
###   @1=2 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='5' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=5 /* SHORTINT meta=0 nullable=1 is_null=0 */
### INSERT INTO `google`.`tb_person`
### SET
###   @1=3 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='10' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=10 /* SHORTINT meta=0 nullable=1 is_null=0 */
# at 179668836
#191123 22:05:11 server id 3232266753  end_log_pos 179668863    Xid = 2788147
COMMIT/*!*/;


########################################################################################################################################
更新事务： Update_rows类型，占用一个gtid_next
事件头以at开头，
事件体的开头：
SET TIMESTAMP=1574513522/*!*/;
BEGIN
/*!*/;
# 中间是事件的内容
事件体的结尾：
COMMIT/*!*/;


# at 179667494
#191123 20:52:02 server id 3232266753  end_log_pos 179667555    GTID    last_committed=557622   sequence_number=557623  rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662578'/*!*/;
# at 179667555
#191123 20:52:02 server id 3232266753  end_log_pos 179667625    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574513522/*!*/;
BEGIN
/*!*/;
# at 179667625
#191123 20:52:02 server id 3232266753  end_log_pos 179667684    Rows_query
# update tb_host set name='10' where id=1
# at 179667684
#191123 20:52:02 server id 3232266753  end_log_pos 179667735    Table_map: `google`.`tb_host` mapped to number 109
# at 179667735
#191123 20:52:02 server id 3232266753  end_log_pos 179667792    Update_rows: table id 109 flags: STMT_END_F
### UPDATE `google`.`tb_host`
### WHERE
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='5' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
### SET
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='10' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
# at 179667792
#191123 20:52:02 server id 3232266753  end_log_pos 179667819    Xid = 2788133
COMMIT/*!*/;

############################################################
一次更新多条记录：Update_rows这个内容与插入的数据量成正比。多个update where set语句


# at 179668863
#191123 22:17:06 server id 3232266753  end_log_pos 179668924    GTID    last_committed=557627   sequence_number=557628  rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662583'/*!*/;
# at 179668924
#191123 22:17:06 server id 3232266753  end_log_pos 179668994    Query   thread_id=557625        exec_time=0     error_code=0
SET TIMESTAMP=1574518626/*!*/;
BEGIN
/*!*/;
# at 179668994
#191123 22:17:06 server id 3232266753  end_log_pos 179669052    Rows_query
# update tb_person set name='glc',age=18
# at 179669052
#191123 22:17:06 server id 3232266753  end_log_pos 179669106    Table_map: `google`.`tb_person` mapped to number 111
# at 179669106
#191123 22:17:06 server id 3232266753  end_log_pos 179669229    Update_rows: table id 111 flags: STMT_END_F
### UPDATE `google`.`tb_person`
### WHERE
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='1' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=1 /* SHORTINT meta=0 nullable=1 is_null=0 */
### SET
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='glc' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=18 /* SHORTINT meta=0 nullable=1 is_null=0 */
### UPDATE `google`.`tb_person`
### WHERE
###   @1=2 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='5' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=5 /* SHORTINT meta=0 nullable=1 is_null=0 */
### SET
###   @1=2 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='glc' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=18 /* SHORTINT meta=0 nullable=1 is_null=0 */
### UPDATE `google`.`tb_person`
### WHERE
###   @1=3 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='10' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=10 /* SHORTINT meta=0 nullable=1 is_null=0 */
### SET
###   @1=3 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='glc' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=18 /* SHORTINT meta=0 nullable=1 is_null=0 */
# at 179669229
#191123 22:17:06 server id 3232266753  end_log_pos 179669256    Xid = 2788148
COMMIT/*!*/;





########################################################################################################################################
删除事务：Delete_rows类型，占用一个gtid_next
事件头以at开头，
事件体开头：
“SET TIMESTAMP=1574513006/*!*/; #该时间戳就是"2019-11-23 21:01:24"的含义。表达了事件发生时间。
BEGIN
/*!*/;” 
# 中间是事件的具体内容:thread_id、Rows_query、Table_map、Delete_rows、Xid
# 即线程id、sql语句、操作的表对象、sql类型、事务结束标志Xid
事件体结尾：“COMMIT/*!*/;”




# at 179667819
#191123 21:01:24 server id 3232266753  end_log_pos 179667880    GTID    last_committed=557623   sequence_number=557624  rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662579'/*!*/;
# at 179667880
#191123 21:01:24 server id 3232266753  end_log_pos 179667950    Query   thread_id=557624        exec_time=0     error_code=0
SET TIMESTAMP=1574514084/*!*/;
BEGIN
/*!*/;
# at 179667950
#191123 21:01:24 server id 3232266753  end_log_pos 179668000    Rows_query
# delete from tb_host where id=1
# at 179668000
#191123 21:01:24 server id 3232266753  end_log_pos 179668051    Table_map: `google`.`tb_host` mapped to number 109
# at 179668051
#191123 21:01:24 server id 3232266753  end_log_pos 179668095    Delete_rows: table id 109 flags: STMT_END_F
### DELETE FROM `google`.`tb_host`
### WHERE
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='10' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
# at 179668095
#191123 21:01:24 server id 3232266753  end_log_pos 179668122    Xid = 2788134
COMMIT/*!*/;

########################################################################################
一次删除多条记录：Update_rows这个内容与插入的数据量成正比。多个delete from where语句

# at 179669256
#191123 22:22:54 server id 3232266753  end_log_pos 179669317    GTID    last_committed=557628   sequence_number=557629  rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662584'/*!*/;
# at 179669317
#191123 22:22:54 server id 3232266753  end_log_pos 179669387    Query   thread_id=557625        exec_time=0     error_code=0
SET TIMESTAMP=1574518974/*!*/;
BEGIN
/*!*/;
# at 179669387
#191123 22:22:54 server id 3232266753  end_log_pos 179669429    Rows_query
# delete from  tb_person
# at 179669429
#191123 22:22:54 server id 3232266753  end_log_pos 179669483    Table_map: `google`.`tb_person` mapped to number 111
# at 179669483
#191123 22:22:54 server id 3232266753  end_log_pos 179669562    Delete_rows: table id 111 flags: STMT_END_F
### DELETE FROM `google`.`tb_person`
### WHERE
###   @1=1 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='glc' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=18 /* SHORTINT meta=0 nullable=1 is_null=0 */
### DELETE FROM `google`.`tb_person`
### WHERE
###   @1=2 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='glc' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=18 /* SHORTINT meta=0 nullable=1 is_null=0 */
### DELETE FROM `google`.`tb_person`
### WHERE
###   @1=3 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='glc' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=18 /* SHORTINT meta=0 nullable=1 is_null=0 */
# at 179669562
#191123 22:22:54 server id 3232266753  end_log_pos 179669589    Xid = 2788149
COMMIT/*!*/;


#######################################################################
###################################
删除数据库
at
GTID
at 251 # drop开始位置为251
end_log_pos 345  Query # drop结束位置是345
第一：GTID事件
第二：Query事件，在这里就是具体的drop事件




# at 190
#191124  0:17:14 server id 3232266753  end_log_pos 251  GTID    last_committed=0        sequence_number=1       rbr_only=no
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662586'/*!*/;
# at 251
#191124  0:17:14 server id 3232266753  end_log_pos 345  Query   thread_id=557625        exec_time=0     error_code=0
SET TIMESTAMP=1574525834/*!*/;
SET @@session.pseudo_thread_id=557625/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549120/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=45/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
drop database google
/*!*/;















#############################################
先插入三条数据，再修改一条数据，再删除一条数据
at  # GTID事件。设置事务隔离级别和GTID_NEXT
GTID
SET TRANSACTION ISOLATION LEVEL
SET @@SESSION.GTID_NEXT
at # Query事件。报告服务端的线程thread_id和事务开始标志BEGIN
Query
at # Rows_query事件。报告表面执行语句
Rows_query
at # Table_map事件。报告操作的表
Table_map
at # Write_rows/Update_rows/Delete_rows事件。报告实际执行语句的具体内容
Write_rows/Update_rows/Delete_rows
at # Xid事件。
Xid # 报告事务结束，每个事务都有唯一的Xid和GTID,在事务提交时，不管是STATEMENT还是ROW格式的binlog，都会在末尾添加一个XID_EVENT事件代表事务的结束。该事件记录了该事务的ID，在MySQL进行崩溃恢复时，根据事务在binlog中的提交情况来决定是否提交存储引擎中状态为prepared的事务。
--xid行给的时间减去最开始的GTID行给的时间就是事务时间。
--xid行给的end_log_pos表示该事务结束的pos位置点。

Xid事件分析：
# at 179670270
#191123 22:29:07 server id 3232266753  end_log_pos 179670297    Xid = 2788151
COMMIT/*!*/;
第一：Xid = 2788151 #表示这是一个事务结束标志，其事务id为2788151，第2788151个事务。
第二：191123 22:29:07 # 表示该事务的结束时间为2019-11-23 22:29:07
第三：at 179670270 # 表示Xid事件的开始位置是179670270
第四：end_log_pos 179670297 # 表示Xid事件的截止位置为179670297
第五：COMMIT/*!*/; # 表示这是一个dml事务类型的commit结束标志。若为/*!*/; 表示ddl语句









事务语句：Rows_query+Table_map+Write_rows/Update_rows/Delete_rows组成，其余部分相同
事务结束标志：Xid

# at 179669589
#191123 22:29:07 server id 3232266753  end_log_pos 179669650    GTID    last_committed=557626   sequence_number=557630  rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662585'/*!*/;
# at 179669650
#191123 22:27:59 server id 3232266753  end_log_pos 179669720    Query   thread_id=557625        exec_time=0     error_code=0
SET TIMESTAMP=1574519279/*!*/;
BEGIN
/*!*/;
# at 179669720
#191123 22:27:59 server id 3232266753  end_log_pos 179669804    Rows_query
# insert into tb_person (name,age) values('1',1),('5',5),('10',10)
# at 179669804
#191123 22:27:59 server id 3232266753  end_log_pos 179669858    Table_map: `google`.`tb_person` mapped to number 111
# at 179669858
#191123 22:27:59 server id 3232266753  end_log_pos 179669932    Write_rows: table id 111 flags: STMT_END_F
### INSERT INTO `google`.`tb_person`
### SET
###   @1=4 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='1' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=1 /* SHORTINT meta=0 nullable=1 is_null=0 */
### INSERT INTO `google`.`tb_person`
### SET
###   @1=5 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='5' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=5 /* SHORTINT meta=0 nullable=1 is_null=0 */
### INSERT INTO `google`.`tb_person`
### SET
###   @1=6 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='10' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=10 /* SHORTINT meta=0 nullable=1 is_null=0 */
# at 179669932
#191123 22:28:32 server id 3232266753  end_log_pos 179670002    Rows_query
# update tb_person set name='glc',age=18 where age=1
# at 179670002
#191123 22:28:32 server id 3232266753  end_log_pos 179670056    Table_map: `google`.`tb_person` mapped to number 111
# at 179670056
#191123 22:28:32 server id 3232266753  end_log_pos 179670118    Update_rows: table id 111 flags: STMT_END_F
### UPDATE `google`.`tb_person`
### WHERE
###   @1=4 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='1' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=1 /* SHORTINT meta=0 nullable=1 is_null=0 */
### SET
###   @1=4 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='glc' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=18 /* SHORTINT meta=0 nullable=1 is_null=0 */
# at 179670118
#191123 22:29:01 server id 3232266753  end_log_pos 179670171    Rows_query
# delete from tb_person where age=5
# at 179670171
#191123 22:29:01 server id 3232266753  end_log_pos 179670225    Table_map: `google`.`tb_person` mapped to number 111
# at 179670225
#191123 22:29:01 server id 3232266753  end_log_pos 179670270    Delete_rows: table id 111 flags: STMT_END_F
### DELETE FROM `google`.`tb_person`
### WHERE
###   @1=5 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='5' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
###   @3=5 /* SHORTINT meta=0 nullable=1 is_null=0 */
# at 179670270
#191123 22:29:07 server id 3232266753  end_log_pos 179670297    Xid = 2788151
COMMIT/*!*/;


#############################################################
####################
查看binlog日志

show binlog events [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count];


show binlog events in 'mysql-bin.000011'  limit 3330000,10000;




################################################################
                            恢复数据
################################################################



从binlog日志恢复数据
恢复命令的语法格式：
mysqlbinlog mysql-bin.0000xx | mysql -u用户名 -p密码 数据库名

--------------------------------------------------------
常用参数选项解释：
--start-position=875 起始pos点
--stop-position=954 结束pos点
--start-datetime="2016-9-25 22:01:08" 起始时间点
--stop-datetime="2019-9-25 22:09:46" 结束时间点
--database=zyyshop 指定只恢复zyyshop数据库(一台主机上往往有多个数据库，只限本地log日志)
-------------------------------------------------------- 
不常用选项： 
-u --user=name 连接到远程主机的用户名
-p --password[=name] 连接到远程主机的密码
-h --host=name 从远程主机上获取binlog日志
--read-from-remote-server 从某个MySQL服务器上读取binlog日志
```



## show slave status\G;

```shell
tmp@127.0.0.1 ((none))> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 127.0.0.1
                  Master_User: tmp
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000013
          Read_Master_Log_Pos: 269728976
               Relay_Log_File: relay-bin.000002
                Relay_Log_Pos: 53412101
        Relay_Master_Log_File: mysql-bin.000013
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 269728976
              Relay_Log_Space: 53412294
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 3232266753
                  Master_UUID: ceabbacf-0c77-11ea-b49f-2016d8c96b46
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662588-1662590
            Executed_Gtid_Set: 108cc4a4-0d40-11ea-9598-2016d8c96b66:1-5,
c42216ad-0d37-11ea-b163-2016d8c96b56:1-9,
ceabbacf-0c77-11ea-b49f-2016d8c96b46:1-1662590
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)

ERROR:
No query specified

tmp@127.0.0.1 ((none))>
# 
```



## show master status\G;

```shell
# 在127.0.0.1:3306主库上执行

tmp@127.0.0.1 ((none))> show variables like '%server%';
+---------------------------------+--------------------------------------+
| Variable_name                   | Value                                |
+---------------------------------+--------------------------------------+
| character_set_server            | utf8mb4                              |
| collation_server                | utf8mb4_general_ci                   |
| innodb_ft_server_stopword_table |                                      |
| server_id                       | 3232266753                           |
| server_id_bits                  | 32                                   |
| server_uuid                     | ceabbacf-0c77-11ea-b49f-2016d8c96b46 |
+---------------------------------+--------------------------------------+
6 rows in set (0.01 sec)
# 根据show variables like '%server_uuid%';
# 可以获得当前mysql实例的server_uuid值

tmp@127.0.0.1 ((none))> show master status\G;
*************************** 1. row ***************************
             File: mysql-bin.000013
         Position: 269728976
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: 108cc4a4-0d40-11ea-9598-2016d8c96b66:1-5,
c42216ad-0d37-11ea-b163-2016d8c96b56:1-9,
ceabbacf-0c77-11ea-b49f-2016d8c96b46:1-1662590
1 row in set (0.00 sec)

ERROR:
No query specified

tmp@127.0.0.1 ((none))>

# 根据主库上执行show master status\G;
# Executed_Gtid_Set值表明：每个server_uuid代表一个实例，有多个server_uuid表明这三个实例都曾经当过主库，分别执行的事务个数都确定。在ceabbacf-0c77-11ea-b49f-2016d8c96b46实例上执行了1662590个事务，在c42216ad-0d37-11ea-b163-2016d8c96b56实例上执行了9个事务，在108cc4a4-0d40-11ea-9598-2016d8c96b66实例上执行了5个事务，但是并不知道这些实例之间事务执行的先后顺序，当然同一个实例上的事务肯定是从1开始递增，步长为1.结合该实例上的server_uuid可知道，当前主库实例执行到了ceabbacf-0c77-11ea-b49f-2016d8c96b46:1-1662590这个位置上了。

# 根据File: mysql-bin.000013和Position: 269728976可知：当前写的二进制日志文件名称和位置是mysql-bin.000013:269728976,在文件mysql-bin.000013中有“end_log_pos 269728976”的地方就是这个位置，如下就是截取了mysql-bin.000013日志最后一部分内容：

# at 269728646
#191124 13:00:04 server id 3232266753  end_log_pos 269728707    GTID    last_committed=16       sequence_number=18      rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ceabbacf-0c77-11ea-b49f-2016d8c96b46:1662590'/*!*/;
# at 269728707
#191124 13:00:04 server id 3232266753  end_log_pos 269728776    Query   thread_id=17    exec_time=0     error_code=0
SET TIMESTAMP=1574571604/*!*/;
BEGIN
/*!*/;
# at 269728776
#191124 13:00:04 server id 3232266753  end_log_pos 269728837    Rows_query
# update table_name set name='2' where id=2
# at 269728837
#191124 13:00:04 server id 3232266753  end_log_pos 269728890    Table_map: `apple`.`table_name` mapped to number 108
# at 269728890
#191124 13:00:04 server id 3232266753  end_log_pos 269728949    Update_rows: table id 108 flags: STMT_END_F
### UPDATE `apple`.`table_name`
### WHERE
###   @1=2 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='8888' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
### SET
###   @1=2 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='2' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
# at 269728949
#191124 13:00:04 server id 3232266753  end_log_pos 269728976    Xid = 119
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```





## 跳过gtid

```shell
stop slave ;
set gtid_next='89dfa8a4-cb13-11e6-b504-000c29a879a3:3';
begin;commit;
set gtid_next='89dfa8a4-cb13-11e6-b504-000c29a879a3:4';
begin;commit;
set gtid_next='automatic';
start slave ;
```













