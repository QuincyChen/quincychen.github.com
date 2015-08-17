---
layout: post
title: "Mysql/MariaDB Innodb insert/delete/update 优化"
---

Ubuntu Server 14.04， MariaDB Cluster Server（Innodb）在数据量较大时（360万条数据），出现了时卡时好的现象；由于系统比较庞杂，初步认为是后台大数据量定时计算等拖累；但经过分析后，发现即使没有任何操作的情况下，系统仍然有卡顿的情况。简单记录下，我在做问题分析中的一些步骤：

* top 查看系统负载情况，系统工作比较正常，负载不大
* df -h 查看系统是否有磁盘空间的问题；发现磁盘占用情况不大 (这个要检查，一次系统很慢的时候，各种原因查找，最后发现是集群中一台备份机器磁盘满，很容易忽略)
* 是否存在DNS解析时间过长，若是这个原因，在配置中(etc/mysql/my.cnf)加入： skip-name-resolve
* 进入Mysql终端 (mysql -uroot -p)，查看系统的连接情况，是否存在大量连接，导致连接数不足的问题: show full processlist; 通过这个查看，也可以分析出哪些查询很慢，看他的时间字段
* 若是连接数的问题，应该考虑系统允许的最大连接数以及整体需求是否匹配，考虑增加最大连接数和减少连接的Sleep时间（减少长时间不释放的sleep连接）
* 若不是连接数的问题，根据上述找到的长时间执行的SQL语句；尝试手工分析效率，具体方式：

<pre class="prettyprint lang-cpp">
	mysql> SET PROFILING=1;
	mysql> INSERT INTO foo ($testdata);
	mysql> show profile for QUERY 1;
</pre>
 
通过上述步骤，分析出，系统在执行insert/delete/update语句时非常慢，并且慢的时候话的最长时间是在query end阶段。

优化的方法：

* 首先考虑到该表做过一次大数据量删除，做一次表的优化：
> mysql> optimize table foo;

	完成后，发现没有明显的改善；
* 调整如下参数(/etc/mysql/my.cnf)
> innodb_buffer_pool_size	= 1024M
> 
> innodb_log_buffer_size	= 256M
	完成后，发现也没有明显改进
	
* 考虑到该表索引比较多（共计5个索引，包括一个主索引），这个会影响插入性能；但是这些索引去除后，会导致select count(*)性能下降，而系统中存在该语句，不可行；
* 调整参数innodb_flush_log_at_trx_commit:
> innodb_flush_log_at_trx_commit = 0

效果达到预期，但是这个参数有副作用：

参见:

* [innodb_flush_log_at_trx_commit](http://dev.mysql.com/doc/refman/5.0/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit)
* [8.6.2 Optimizing InnoDB Transaction Management](http://dev.mysql.com/doc/refman/5.0/en/optimizing-innodb-transaction-management.html)
<pre class="prettyprint lang-cpp">

英文原文及对应翻译如下：

innodb_flush_log_at_trx_commit
Command-Line Format     Cinnodb_flush_log_at_trx_commit[=#]
Config-File Format     innodb_flush_log_at_trx_commit
Option Sets Variable     Yes, innodb_flush_log_at_trx_commit
Variable Name     innodb_flush_log_at_trx_commit
Variable Scope     Global
Dynamic Variable     Yes
Permitted Values
Type     numeric
Default     1
Valid Values     0, 1, 2

If the value of innodb_flush_log_at_trx_commit is 0, the log buffer is written out to the log file once per second and the flush to disk operation is performed on the log file, but nothing is done at a transaction commit. When the value is 1 (the default), the log buffer is written out to the log file at each transaction commit and the flush to disk operation is performed on the log file. When the value is 2, the log buffer is written out to the file at each commit, but the flush to disk operation is not performed on it. However, the flushing on the log file takes place once per second also when the value is 2. Note that the once-per-second flushing is not 100% guaranteed to happen every second, due to process scheduling issues.

如果innodb_flush_log_at_trx_commit设置为0，log buffer将每秒一次地写入log file中，并且log file的flush(刷到磁盘)操作同时进行；但是，这种模式下，在事务提交的时候，不会有任何动作。如果 innodb_flush_log_at_trx_commit设置为1(默认值)，log buffer每次事务提交都会写入log file，并且，flush刷到磁盘中去。如果innodb_flush_log_at_trx_commit设置为2，log buffer在每次事务提交的时候都会写入log file，但是，flush(刷到磁盘)操作并不会同时进行。这种模式下，MySQL会每秒一次地去做flush(刷到磁盘)操作。注意：由于进程调度策 略问题，这个“每秒一次的flush(刷到磁盘)操作”并不是保证100%的“每秒”。

The default value of 1 is the value required for ACID compliance. You can achieve better performance by setting the value different from 1, but then you can lose at most one second worth of transactions in a crash. With a value of 0, any mysqld process crash can erase the last second of transactions. With a value of 2, then only an operating system crash or a power outage can erase the last second of transactions. However, InnoDB’s crash recovery is not affected and thus crash recovery does work regardless of the value.

默认值1是为了ACID (atomicity, consistency, isolation, durability)原子性，一致性，隔离性和持久化的考虑。如果你不把innodb_flush_log_at_trx_commit设置为1，你将 获得更好的性能，但是，你在系统崩溃的情况，可能会丢失最多一秒钟的事务数据。当你把innodb_flush_log_at_trx_commit设置 为0，mysqld进程的崩溃会导致上一秒钟所有事务数据的丢失。如果你把innodb_flush_log_at_trx_commit设置为2，只有 在操作系统崩溃或者系统掉电的情况下，上一秒钟所有事务数据才可能丢失。(下面的这句话到底是针对 innodb_flush_log_at_trx_commit为2说的，还是针对前面这一整段说的，我就搞不清楚了，下次问问编写这一段文档的 MySQL的人去。感觉是针对整段的：就是说InnoDB的crash recovery会利用log file来恢复数据文件，跟innodb_flush_log_at_trx_commit的值没有关系，管你这个值怎么设置的，我从log file拿到多少数据，就恢复多少数据。)InnoDB的crash recovery崩溃恢复机制并不受这个值的影响，不管这个值设置为多少，crash recovery崩溃恢复机制都会工作。

For the greatest possible durability and consistency in a replication setup using InnoDB with transactions, use innodb_flush_log_at_trx_commit = 1 and sync_binlog = 1 in your master server my.cnf file.

为了在使用InnoDB事务的搭建复制环境中，达到最大的持久化和一致性，你需要在你的master主机的my.cnf中设置innodb_flush_log_at_trx_commit = 1并且设置sync_binlog = 1。

Caution

Many operating systems and some disk hardware fool the flush-to-disk operation. They may tell mysqld that the flush has taken place, even though it has not. Then the durability of transactions is not guaranteed even with the setting 1, and in the worst case a power outage can even corrupt the InnoDB database. Using a battery-backed disk cache in the SCSI disk controller or in the disk itself speeds up file flushes, and makes the operation safer. You can also try using the Unix command hdparm to disable the caching of disk writes in hardware caches, or use some other command specific to the hardware vendor.

注意：
很多操作系统和一些磁盘硬件系统并不会真正的做flush-to-disk刷新到磁盘的这个操作。他们即使并没有真正刷到磁盘也会告诉mysqld说 flush刷新到磁盘的操作已经完成了。这样的话，即使innodb_flush_log_at_trx_commit设置为1，也不能保证事务的持久 化，最糟的情况下，一个主机掉电，就有可能导致InnoDB数据库崩溃。你可以考虑在SCSI磁盘控制器里面或者磁盘本身中，使用带蓄电池后备电源的磁盘 缓存disk cache，来提高文件刷新操作的速度，使得这个操作更加安全。你同样可以尝试使用Unix的hdparm命令来阻止硬件缓存hardware cache的写磁盘缓存操作，或者使用其他硬件提供商hardware vendor提供的命令来避免写磁盘缓存。

这里既然提到了sync_binlog就顺便把它也翻译一下。
sync_binlog
Command-Line Format     Csync-binlog=#
Config-File Format     sync_binlog
Option Sets Variable     Yes, sync_binlog
Variable Name     sync_binlog
Variable Scope     Global
Dynamic Variable     Yes
Permitted Values
Platform Bit Size     32
Type     numeric
Default     0
Range     0-4294967295
Permitted Values
Platform Bit Size     64
Type     numeric
Default     0
Range     0-18446744073709547520

If the value of this variable is greater than 0, the MySQL server synchronizes its binary log to disk (using fdatasync()) after every sync_binlog writes to the binary log. There is one write to the binary log per statement if autocommit is enabled, and one write per transaction otherwise. The default value of sync_binlog is 0, which does no synchronizing to disk ― in this case, the server relies on the operating system to flush the binary log’s contents from to time as for any other file. A value of 1 is the safest choice because in the event of a crash you lose at most one statement or transaction from the binary log. However, it is also the slowest choice (unless the disk has a battery-backed cache, which makes synchronization very fast).

当sync_binlog变量设置为大于0的值时，MySQL在每次“sync_binlog”这么多次写二进制日志binary log时，会使用fdatasync()函数将它的写二进制日志binary log同步到磁盘中去。如果启用了autocommit，那么每一个语句statement就会有一次写操作；否则每个事务对应一个写操作。 sync_binlog的默认值是0，这种模式下，MySQL不会同步到磁盘中去。这样的话，MySQL依赖操作系统来刷新二进制日志binary log，就像操作系统刷其他文件的机制一样。当sync_binlog变量设置为1是最安全的，因为在crash崩溃的情况下，你的二进制日志 binary log只有可能丢失最多一个语句或者一个事务。但是，这也是最慢的一种方式（除非磁盘有使用带蓄电池后备电源的缓存cache，使得同步到磁盘的操作非常 快）。

找了一下man fdatasync:
fdatasync() flushes all data buffers of a file to disk (before the system call returns).  It resembles fsync() but is not required to update the metadata such as access time.
fdatasync() (在系统调用system call返回前)将文件中所有的数据缓存区data buffers都flush刷到磁盘中去。它类似于fsync()函数，但是它不会更新元数据metadata：比如最后访问时间等。
</pre>

综合考虑上述因素，比较好的做法：

* 考虑磁盘性能问题，优化磁盘的I/O性能
* 关闭AutoCommit，然后一次执行一批插入更新等语句，最后进行Commit，这样减少日志写次数，提高效率【需开发人员修改程序】
* 考虑将innodb_flush_log_at_trx_commit设置为2或0，虽然这个有缺陷，但是可以有效解决系统卡顿问题

如果你有比较好的方法，请不啬赐教。


以下是一些参考链接：

* [why-is-mysql-innodb-insert-so-slow](http://stackoverflow.com/questions/9819271/why-is-mysql-innodb-insert-so-slow)
* [Why do MySQL InnoDB inserts / updates on large tables get very slow when there are a few indexes?](http://stackoverflow.com/questions/2222861/why-do-mysql-innodb-inserts-updates-on-large-tables-get-very-slow-when-there-a)
* [InnoDB inserts very slow and slowing down](http://stackoverflow.com/questions/9114209/innodb-inserts-very-slow-and-slowing-down)
* [innodb_flush_log_at_trx_commit](http://www.cnblogs.com/whiteyun/archive/2011/12/01/2270132.html)
* [Mysql innodb insert into 性能优化](http://www.360doc.com/content/12/0308/10/834950_192667954.shtml)
* [Mysql update takes a long time on “end” state](http://stackoverflow.com/questions/21556900/mysql-update-takes-a-long-time-on-end-state)
* [“query end” step very long at random times](http://stackoverflow.com/questions/6937443/query-end-step-very-long-at-random-times)

