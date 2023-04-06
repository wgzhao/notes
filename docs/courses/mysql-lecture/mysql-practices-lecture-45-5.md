---
title: 《MySQL 实战45讲》节选第五部分
description: 这篇摘要内容节选自 “丁奇” 在极客时间的 《MySQL实战45讲》的内容，这是第五部分
tags: ["mysql", "tuning"]
---

# 《MySQL 实战45讲》节选第五部分

[Source](http://learn.lianglianglee.com/%E6%9E%81%E5%AE%A2%E6%97%B6%E9%97%B4/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2.md "Permalink to MySQL实战45讲.md")

以下内容节选自 “丁奇” 在极客时间的 《MySQL实战45讲》的内容，这是第五部分

## 40 insert语句的锁为什么这么多？

在上一篇文章中，我提到 MySQL 对自增主键锁做了优化，尽量在申请到自增 id 以后，就释放自增锁。

因此，insert 语句是一个很轻量的操作。不过，这个结论对于“普通的 insert 语句”才有效。也就是说，还有些 insert 语句是属于“特殊情况”的，在执行过程中需要给其他资源加锁，或者无法在申请到自增 id 以后就立马释放自增锁。

那么，今天这篇文章，我们就一起来聊聊这个话题。

### insert … select 语句

我们先从昨天的问题说起吧。表 t 和 t2 的表结构、初始化数据语句如下，今天的例子我们还是针对这两个表展开。

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB; 
insert into t values(null, 1,1);
insert into t values(null, 2,2);
insert into t values(null, 3,3);
insert into t values(null, 4,4); 
create table t2 like t;
```

现在，我们一起来看看为什么在可重复读隔离级别下，`binlog_format=statement` 时执行：

```sql
insert into t2(c,d) select c,d from t;
```

这个语句时，需要对表 t 的所有行和间隙加锁呢？

其实，这个问题我们需要考虑的还是日志和数据的一致性。我们看下这个执行序列：

- session A: `insert into t values(-1, -1, -1)`
- session B: `insert into t2(c, d) select c, d from t`

实际的执行效果是，如果 session B 先执行，由于这个语句对表 t 主键索引加了 (-∞,1] 这个 next-key lock，会在语句执行完成后，才允许 session A 的 insert 语句执行。

但如果没有锁的话，就可能出现 session B 的 insert 语句先执行，但是后写入 binlog 的情况。于是，在 `binlog_format=statement` 的情况下，binlog 里面就记录了这样的语句序列：

```sql
insert into t values(-1,-1,-1);
insert into t2(c,d) select c,d from t;
```

这个语句到了备库执行，就会把 id=-1 这一行也写到表 t2 中，出现主备不一致。

### insert 循环写入

当然了，执行 `insert … select` 的时候，对目标表也不是锁全表，而是只锁住需要访问的资源。

如果现在有这么一个需求：要往表 t2 中插入一行数据，这一行的 c 值是表 t 中 c 值的最大值加 1。

此时，我们可以这么写这条 SQL 语句 ：

```sql
insert into t2(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
```

这个语句的加锁范围，就是表 t 索引 c 上的 (3,4] 和 (4,supremum] 这两个 next-key lock，以及主键索引上 id=4 这一行。

它的执行流程也比较简单，从表 t 中按照索引 c 倒序，扫描第一行，拿到结果写入到表 t2 中。

因此整条语句的扫描行数是 1。

那么，如果我们是要把这样的一行数据插入到表 t 中的话：

```sql
insert into t(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
```

语句的执行流程是怎样的？扫描行数又是多少呢？

这时候，我们再看慢查询日志就会发现不对了。

```sql
# Query_time: 0.001429  Lock_time: 0.000213 Rows_sent: 0  Rows_examined: 5
SET timestamp=1675306887;
insert into t(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
```

可以看到，这时候的 Rows\_examined 的值是 5。

我在前面的文章中提到过，希望你都能够学会用 explain 的结果来“脑补”整条语句的执行过程。今天，我们就来一起试试。

如图 4 所示就是这条语句的 explain 结果。

```sql
mysql> explain insert into t(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+--------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+--------------------------------------+
|  1 | INSERT      | t     | NULL       | ALL   | NULL          | NULL | NULL    | NULL | NULL |     NULL | NULL                                 |
|  1 | SIMPLE      | t     | NULL       | index | NULL          | c    | 5       | NULL |    1 |   100.00 | Backward index scan; Using temporary |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+--------------------------------------+
```

从 Extra 字段可以看到“Using temporary”字样，表示这个语句用到了临时表。也就是说，执行过程中，需要把表 t 的内容读出来，写入临时表。

图中 rows 显示的是 1，我们不妨先对这个语句的执行流程做一个猜测：如果说是把子查询的结果读出来（扫描 1 行），写入临时表，然后再从临时表读出来（扫描 1 行），写回表 t 中。那么，这个语句的扫描行数就应该是 2，而不是 5。

所以，这个猜测不对。实际上，Explain 结果里的 rows=1 是因为受到了 limit 1 的影响。

从另一个角度考虑的话，我们可以看看 InnoDB 扫描了多少行。如图 5 所示，是在执行这个语句前后查看 `Innodb_rows_read` 的结果。

```sql
mysql> show status like 'Innodb_rows_read';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| Innodb_rows_read | 221600 |
+------------------+--------+
1 row in set (0.01 sec)

mysql>  insert into t(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
Query OK, 1 row affected, 2 warnings (0.04 sec)
Records: 1  Duplicates: 0  Warnings: 2

mysql> show status like 'Innodb_rows_read';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| Innodb_rows_read | 221601 |
+------------------+--------+
1 row in set (0.00 sec)
```

这样，我们就把整个执行过程理清楚了：

1. 创建临时表，表里有两个字段 c 和 d。
2. 按照索引 c 扫描表 t，依次取 c=4、3、2、1，然后回表，读到 c 和 d 的值写入临时表。这时，Rows\_examined=4。
3. 由于语义里面有 limit 1，所以只取了临时表的第一行，再插入到表 t 中。这时，Rows\_examined 的值加 1，变成了 5。

也就是说，这个语句会导致在表 t 上做全表扫描，并且会给索引 c 上的所有间隙都加上共享的 next-key lock。所以，这个语句执行期间，其他事务不能在这个表上插入数据。

至于这个语句的执行为什么需要临时表，原因是这类一边遍历数据，一边更新数据的情况，如果读出来的数据直接写回原表，就可能在遍历过程中，读到刚刚插入的记录，新插入的记录如果参与计算逻辑，就跟语义不符。

由于实现上这个语句没有在子查询中就直接使用 limit 1，从而导致了这个语句的执行需要遍历整个表 t。它的优化方法也比较简单，就是用前面介绍的方法，先 insert into 到临时表 temp\_t，这样就只需要扫描一行；然后再从表 temp\_t 里面取出这行数据插入表 t1。

当然，由于这个语句涉及的数据量很小，你可以考虑使用内存临时表来做这个优化。使用内存临时表优化时，语句序列的写法如下：

```sql
create temporary table temp_t(c int,d int) engine=memory;
insert into temp_t  (select c+1, d from t force index(c) order by c desc limit 1);
insert into t select * from temp_t;
drop table temp_t;
```

### 小结

今天这篇文章，我和你介绍了几种特殊情况下的 insert 语句。

insert … select 是很常见的在两个表之间拷贝数据的方法。你需要注意，在可重复读隔离级别下，这个语句会给 select 的表里扫描到的记录和间隙加读锁。

而如果 insert 和 select 的对象是同一个表，则有可能会造成循环写入。这种情况下，我们需要引入用户临时表来做优化。

insert 语句如果出现唯一键冲突，会在冲突的唯一值上加共享的 next-key lock(S 锁)。因此，碰到由于唯一键约束导致报错后，要尽快提交或回滚事务，避免加锁时间过长。



## 41 怎么最快地复制一张表？

我在上一篇文章最后，给你留下的问题是怎么在两张表中拷贝数据。如果可以控制对源表的扫描行数和加锁范围很小的话，我们简单地使用 `insert … select` 语句即可实现。

当然，为了避免对源表加读锁，更稳妥的方案是先将数据写到外部文本文件，然后再写回目标表。这时，有两种常用的方法。接下来的内容，我会和你详细展开一下这两种方法。

为了便于说明，我还是先创建一个表 db1.t，并插入 1000 行数据，同时创建一个相同结构的表 db2.t。

```sql
create database db1;
use db1; 
create table t(id int primary key, a int, b int, index(a))engine=innodb;
delimiter ;;
  create procedure idata()
  begin
    declare i int;
    set i=1;
    while(i<=1000)do
      insert into t values(i,i,i);
      set i=i+1;
    end while;
  end;;
delimiter ;
call idata(); 
create database db2;
create table db2.t like db1.t
```

假设，我们要把 db1.t 里面 a\>900 的数据行导出来，插入到 db2.t 中。

### mysqldump 方法

一种方法是，使用 mysqldump 命令将数据导出成一组 INSERT 语句。你可以使用下面的命令：

```sql
$ mysqldump -h$host -P$port -u$user --add-locks=0 --no-create-info --single-transaction  \
  --set-gtid-purged=OFF db1 t --where="a>900" --result-file=/tmp/t.sql
```

把结果输出到临时文件。

这条命令中，主要参数含义如下：

1. `–-single-transaction` 的作用是，在导出数据的时候不需要对表 db1.t 加表锁，而是使用 START TRANSACTION WITH CONSISTENT SNAPSHOT 的方法；
2. `–-add-locks` 设置为 0，表示在输出的文件结果里，不增加 ` LOCK TABLES t WRITE;` ；
3. `–-no-create-info` 的意思是，不需要导出表结构；
4. `-–set-gtid-purged=off` 表示的是，不输出跟 GTID 相关的信息；
5. `-–result-file` 指定了输出文件的路径。

可以看到，一条 INSERT 语句里面会包含多个 value 对，这是为了后续用这个文件来写入数据的时候，执行速度可以更快。

如果你希望生成的文件中一条 INSERT 语句只插入一行数据的话，可以在执行 mysqldump 命令时，加上参数 `–-skip-extended-insert`。

然后，你可以通过下面这条命令，将这些 INSERT 语句放到 db2 库里去执行。

```sql
mysql -h127.0.0.1 -P13000  -uroot db2 -e "source /client_tmp/t.sql"
```

需要说明的是，source 并不是一条 SQL 语句，而是一个客户端命令。mysql 客户端执行这个命令的流程是这样的：

1. 打开文件，默认以分号为结尾读取一条条的 SQL 语句；
2. 将 SQL 语句发送到服务端执行。

也就是说，服务端执行的并不是这个 `source t.sql` 语句，而是 INSERT 语句。所以，不论是在慢查询日志（slow log），还是在 binlog，记录的都是这些要被真正执行的 INSERT 语句。

### 导出 CSV 文件

另一种方法是直接将结果导出成.csv 文件。MySQL 提供了下面的语法，用来将查询结果导出到服务端本地目录：

```sql
mysql> select * from db1.t where a>900 into outfile '/server_tmp/t.csv';
```

我们在使用这条语句时，需要注意如下几点。

1. 这条语句会将结果保存在服务端。如果你执行命令的客户端和 MySQL 服务端不在同一个机器上，客户端机器的临时目录下是不会生成 t.csv 文件的。
2. into outfile 指定了文件的生成位置（`/server_tmp/`），这个位置必须受参数 `secure_file_priv` 的限制。参数 `secure_file_priv` 的可选值和作用分别是： 
  * 如果设置为 empty，表示不限制文件生成的位置，这是不安全的设置；
  * 如果设置为一个表示路径的字符串，就要求生成的文件只能放在这个指定的目录，或者它的子目录；
  * 如果设置为 NULL，就表示禁止在这个 MySQL 实例上执行 `select … into outfile` 操作。
3. 这条命令不会帮你覆盖文件，因此你需要确保 `/server_tmp/t.csv` 这个文件不存在，否则执行语句时就会因为有同名文件的存在而报错。
4. 这条命令生成的文本文件中，原则上一个数据行对应文本文件的一行。但是，如果字段中包含换行符，在生成的文本中也会有换行符。不过类似换行符、制表符这类符号，前面都会跟上 `“\”`这个转义符，这样就可以跟字段之间、数据行之间的分隔符区分开。

得到.csv 导出文件后，你就可以用下面的 load data 命令将数据导入到目标表 db2.t 中。

```sql
mysql> load data infile '/server_tmp/t.csv' into table db2.t;
```

这条语句的执行流程如下所示。

1. 打开文件 `/server_tmp/t.csv`，以制表符 ( `\t`) 作为字段间的分隔符，以换行符（ `\n`）作为记录之间的分隔符，进行数据读取；
2. 启动事务。
3. 判断每一行的字段数与表 db2.t 是否相同： 
  * 若不相同，则直接报错，事务回滚；
  * 若相同，则构造成一行，调用 InnoDB 引擎接口，写入到表中。
4. 重复步骤 3，直到 `/server_tmp/t.csv` 整个文件读入完成，提交事务。

你可能有一个疑问，**如果 binlog\_format=statement，这个 load 语句记录到 binlog 里以后，怎么在备库重放呢？**

由于 `/server_tmp/t.csv` 文件只保存在主库所在的主机上，如果只是把这条语句原文写到 binlog 中，在备库执行的时候，备库的本地机器上没有这个文件，就会导致主备同步停止。

所以，这条语句执行的完整流程，其实是下面这样的。

1. 主库执行完成后，将 `/server_tmp/t.csv` 文件的内容直接写到 binlog 文件中。
2. 往 binlog 文件中写入语句 `load data local infile '/tmp/SQL_LOAD_MB-1-0' INTO TABLE db2.t`。
3. 把这个 binlog 日志传到备库。
4. 备库的 apply 线程在执行这个事务日志时： 
   1. 先将 binlog 中 t.csv 文件的内容读出来，写入到本地临时目录 `/tmp/SQL_LOAD_MB-1-0 `中； 
   2.  再执行 load data 语句，往备库的 db2.t 表中插入跟主库相同的数据。


注意，这里备库执行的 load data 语句里面，多了一个 `local`。它的意思是“将执行这条命令的客户端所在机器的本地文件 `/tmp/SQL_LOAD_MB-1-0` 的内容，加载到目标表 db2.t 中”。

也就是说，**load data 命令有两种用法**：

1. 不加 `local`，是读取服务端的文件，这个文件必须在 `secure_file_priv` 指定的目录或子目录下；
2. 加上 `local`，读取的是客户端的文件，只要 mysql 客户端有访问这个文件的权限即可。这时候，MySQL 客户端会先把本地文件传给服务端，然后执行上述的 load data 流程。

另外需要注意的是，**select …into outfile 方法不会生成表结构文件**, 所以我们导数据时还需要单独的命令得到表结构定义。mysqldump 提供了一个`-–tab` 参数，可以同时导出表结构定义文件和 csv 数据文件。这条命令的使用方法如下：

```sql
mysqldump -h$host -P$port -u$user ---single-transaction  --set-gtid-purged=OFF db1 t --where="a>900" --tab=$secure_file_priv
```

这条命令会在 `$secure_file_priv` 定义的目录下，创建一个 `t.sql` 文件保存建表语句，同时创建一个 `t.txt` 文件保存 CSV 数据。

### 物理拷贝方法

前面我们提到的 mysqldump 方法和导出 CSV 文件的方法，都是逻辑导数据的方法，也就是将数据从表 db1.t 中读出来，生成文本，然后再写入目标表 db2.t 中。

你可能会问，有物理导数据的方法吗？比如，直接把 db1.t 表的 `.frm` 文件和 `.ibd` 文件拷贝到 db2 目录下，是否可行呢？

答案是不行的。

因为，一个 InnoDB 表，除了包含这两个物理文件外，还需要在数据字典中注册。直接拷贝这两个文件的话，因为数据字典中没有 db2.t 这个表，系统是不会识别和接受它们的。

不过，在 MySQL 5.6 版本引入了**可传输表空间**(transportable tablespace) 的方法，可以通过导出 + 导入表空间的方式，实现物理拷贝表的功能。

假设我们现在的目标是在 db1 库下，复制一个跟表 t 相同的表 r，具体的执行步骤如下：

1. 执行 `create table r like t`，创建一个相同表结构的空表；
2. 执行 `alter table r discard tablespace`，这时候 r.ibd 文件会被删除；
3. 执行 `flush table t for export`，这时候 db1 目录下会生成一个 t.cfg 文件；
4. 在 db1 目录下执行 `cp t.cfg r.cfg; cp t.ibd r.ibd；`这两个命令（这里需要注意的是，拷贝得到的两个文件，MySQL 进程要有读写权限）；
5. 执行 `unlock tables`，这时候 t.cfg 文件会被删除；
6. 执行 `alter table r import tablespace`，将这个 `r.ibd` 文件作为表 r 的新的表空间，由于这个文件的数据内容和 t.ibd 是相同的，所以表 r 中就有了和表 t 相同的数据。

至此，拷贝表数据的操作就完成了。这个流程的执行过程图如下：

关于拷贝表的这个流程，有以下几个注意点：

1. 在第 3 步执行完 `flsuh table` 命令之后，`db1.t` 整个表处于只读状态，直到执行 `unlock tables` 命令后才释放读锁；
2. 在执行 `import tablespace` 的时候，为了让文件里的表空间 id 和数据字典中的一致，会修改 r.ibd 的表空间 id。而这个表空间 id 存在于每一个数据页中。因此，如果是一个很大的文件（比如 TB 级别），每个数据页都需要修改，所以你会看到这个 import 语句的执行是需要一些时间的。当然，如果是相比于逻辑导入的方法，import 语句的耗时是非常短的。

### 小结

今天这篇文章，我和你介绍了三种将一个表的数据导入到另外一个表中的方法。

我们来对比一下这三种方法的优缺点。

1. 物理拷贝的方式速度最快，尤其对于大表拷贝来说是最快的方法。如果出现误删表的情况，用备份恢复出误删之前的临时库，然后再把临时库中的表拷贝到生产库上，是恢复数据最快的方法。但是，这种方法的使用也有一定的局限性： 
  * 必须是全表拷贝，不能只拷贝部分数据；
  * 需要到服务器上拷贝数据，在用户无法登录数据库主机的场景下无法使用；
  * 由于是通过拷贝物理文件实现的，源表和目标表都是使用 InnoDB 引擎时才能使用。
2. 用 mysqldump 生成包含 INSERT 语句文件的方法，可以在 where 参数增加过滤条件，来实现只导出部分数据。这个方式的不足之一是，不能使用 join 这种比较复杂的 where 条件写法。
3. 用 `select … into outfile` 的方法是最灵活的，支持所有的 SQL 写法。但，这个方法的缺点之一就是，每次只能导出一张表的数据，而且表结构也需要另外的语句单独备份。

后两种方式都是逻辑备份方式，是可以跨引擎使用的。



## 42 grant之后要跟着flush privileges吗？

在 MySQL 里面，grant 语句是用来给用户赋权的。不知道你有没有见过一些操作文档里面提到，grant 之后要马上跟着执行一个 flush privileges 命令，才能使赋权语句生效。我最开始使用 MySQL 的时候，就是照着一个操作文档的说明按照这个顺序操作的。

那么，grant 之后真的需要执行 flush privileges 吗？如果没有执行这个 flush 命令的话，赋权语句真的不能生效吗？

接下来，我就先和你介绍一下 grant 语句和 flush privileges 语句分别做了什么事情，然后再一起来分析这个问题。

为了便于说明，我先创建一个用户：

```sql
create user 'ua'@'%' identified by 'pa';
```

这条命令做了两个动作：

1. 磁盘上，往 `mysql.user` 表里插入一行，由于没有指定权限，所以这行数据上所有表示权限的字段的值都是 N；
2. 内存里，往数组 `acl_users` 里插入一个 `acl_user` 对象，这个对象的 access 字段值为 0。

图 1 就是这个时刻用户 ua 在 user 表中的状态。

```sql
mysql> select * from mysql.user where user='ua'\G;
*************************** 1. row ***************************
                    Host: %
                    User: ua
             Select_priv: N
             Insert_priv: N
             Update_priv: N
             Delete_priv: N
             Create_priv: N
               Drop_priv: N
             Reload_priv: N
           Shutdown_priv: N
            Process_priv: N
               File_priv: N
              Grant_priv: N
         References_priv: N
              Index_priv: N
              Alter_priv: N
            Show_db_priv: N
              Super_priv: N
   Create_tmp_table_priv: N
        Lock_tables_priv: N
            Execute_priv: N
         Repl_slave_priv: N
        Repl_client_priv: N
        Create_view_priv: N
          Show_view_priv: N
     Create_routine_priv: N
      Alter_routine_priv: N
        Create_user_priv: N
              Event_priv: N
            Trigger_priv: N
  Create_tablespace_priv: N
                ssl_type:
              ssl_cipher:
             x509_issuer:
            x509_subject:
           max_questions: 0
             max_updates: 0
         max_connections: 0
    max_user_connections: 0
                  plugin: caching_sha2_password
   authentication_string: $A$005$
%P:eJ8= )7_{P=.1QQ3BFIVgLLcQ6MWmI61WcjMGnKMpgk9RGdANON2Pn6NB
        password_expired: N
   password_last_changed: 2023-02-02 11:33:19
       password_lifetime: NULL
          account_locked: N
        Create_role_priv: N
          Drop_role_priv: N
  Password_reuse_history: NULL
     Password_reuse_time: NULL
Password_require_current: NULL
         User_attributes: NULL
```

在 MySQL 中，用户权限是有不同的范围的。接下来，我就按照用户权限范围从大到小的顺序依次和你说明。

### 全局权限

全局权限，作用于整个 MySQL 实例，这些权限信息保存在 mysql 库的 user 表里。如果我要给用户 ua 赋一个最高权限的话，语句是这么写的：

```sql
grant all privileges on *.* to 'ua'@'%' with grant option;
```

这个 grant 命令做了两个动作：

1. 磁盘上，将 `mysql.user` 表里，用户 `'ua'@'%'`这一行的所有表示权限的字段的值都修改为 `Y`；
2. 内存里，从数组 `acl_users` 中找到这个用户对应的对象，将 access 值（权限位）修改为二进制的“全 1”。

在这个 grant 命令执行完成后，如果有新的客户端使用用户名 ua 登录成功，MySQL 会为新连接维护一个线程对象，然后从 `acl_users` 数组里查到这个用户的权限，并将权限值拷贝到这个线程对象中。之后在这个连接中执行的语句，所有关于全局权限的判断，都直接使用线程对象内部保存的权限位。

基于上面的分析我们可以知道：

1. grant 命令对于全局权限，同时更新了磁盘和内存。命令完成后即时生效，接下来新创建的连接会使用新的权限。
2. 对于一个已经存在的连接，它的全局权限不受 grant 命令的影响。

需要说明的是，**一般在生产环境上要合理控制用户权限的范围**。我们上面用到的这个 grant 语句就是一个典型的错误示范。如果一个用户有所有权限，一般就不应该设置为所有 IP 地址都可以访问。

如果要回收上面的 grant 语句赋予的权限，你可以使用下面这条命令：

```sql
revoke all privileges on *.* from 'ua'@'%';
```

这条 revoke 命令的用法与 grant 类似，做了如下两个动作：

1. 磁盘上，将 `mysql.user` 表里，用户 `'ua'@'%'` 这一行的所有表示权限的字段的值都修改为 `N`；
2. 内存里，从数组 acl\_users 中找到这个用户对应的对象，将 access 的值修改为 0。

### db 权限

除了全局权限，MySQL 也支持库级别的权限定义。如果要让用户 ua 拥有库 db1 的所有权限，可以执行下面这条命令：

```sql
grant all privileges on db1.* to 'ua'@'%' with grant option;
```

基于库的权限记录保存在 mysql.db 表中，在内存里则保存在数组 acl\_dbs 中。这条 grant 命令做了如下两个动作：

1. 磁盘上，往 `mysql.db` 表中插入了一行记录，所有权限位字段设置为 `Y`；
2. 内存里，增加一个对象到数组 `acl_dbs` 中，这个对象的权限位为“全 1”。

图 2 就是这个时刻用户 ua 在 db 表中的状态。

```sql
mysql> select * from mysql.db where user='ua'\G;
*************************** 1. row ***************************
                 Host: %
                   Db: db1
                 User: ua
          Select_priv: Y
          Insert_priv: Y
          Update_priv: Y
          Delete_priv: Y
          Create_priv: Y
            Drop_priv: Y
           Grant_priv: Y
      References_priv: Y
           Index_priv: Y
           Alter_priv: Y
Create_tmp_table_priv: Y
     Lock_tables_priv: Y
     Create_view_priv: Y
       Show_view_priv: Y
  Create_routine_priv: Y
   Alter_routine_priv: Y
         Execute_priv: Y
           Event_priv: Y
         Trigger_priv: Y
```

每次需要判断一个用户对一个数据库读写权限的时候，都需要遍历一次 `acl_dbs` 数组，根据 user、host 和 db 找到匹配的对象，然后根据对象的权限位来判断。

也就是说，grant 修改 db 权限的时候，是同时对磁盘和内存生效的。

grant 操作对于已经存在的连接的影响，在全局权限和基于 db 的权限效果是不同的。接下来，我们做一个对照试验来分别看一下。

|      | session A                                                    | session B                                                    |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| T1   | connect(root, root);<br />create databse db1;<br />create user 'ua'@'%' identified by 'pa';<br />grant super on \*.\* to 'ua'@'%';<br />grant all privileges on db1.* to 'ua'@'%'; |                                                              |
| T2   |                                                              | connect(ua,pa);<br />set global sync_binlog=1;<br />(Query OK)<br />create table db1.t(c int);<br />(Query OK) |
| T3   | revoke super on *.* from 'ua'@'%';                           |                                                              |
| T4   |                                                              | set global sync_binlog=1;<br />(Query OK)<br />alter table db1.t engine=innodb;<br />(Query OK) |
| T5   | revoke all privileges on db1.* from 'ua'@'%';                |                                                              |
| T6   |                                                              | set global sync_binlog=1;<br />(Query OK)<br />alter table db1.t engine=innodb;<br />(ALTER command denied) |

需要说明的是，图中 `set global sync_binlog` 这个操作是需要 super 权限的。

可以看到，虽然用户 ua 的 super 权限在 T3 时刻已经通过 revoke 语句回收了，但是在 T4 时刻执行 set global 的时候，权限验证还是通过了。这是因为 super 是全局权限，这个权限信息在线程对象中，而 revoke 操作影响不到这个线程对象。

而在 T5 时刻去掉 ua 对 db1 库的所有权限后，在 T6 时刻 session B 再操作 db1 库的表，就会报错“权限不足”。这是因为 `acl_dbs` 是一个全局数组，所有线程判断 db 权限都用这个数组，这样 revoke 操作马上就会影响到 session B。

### 表权限和列权限

除了 db 级别的权限外，MySQL 支持更细粒度的表权限和列权限。其中，表权限定义存放在表 `mysql.tables_priv` 中，列权限定义存放在表 `mysql.columns_priv` 中。这两类权限，组合起来存放在内存的 hash 结构 `column_priv_hash` 中。

这两类权限的赋权命令如下：

```sql
create table db1.t1(id int, a int); 
grant all privileges on db1.t1 to 'ua'@'%' with grant option;
GRANT SELECT(id), INSERT (id,a) ON mydb.mytbl TO 'ua'@'%' with grant option;
```

跟 db 权限类似，这两个权限每次 grant 的时候都会修改数据表，也会同步修改内存中的 hash 结构。因此，对这两类权限的操作，也会马上影响到已经存在的连接。

看到这里，你一定会问，看来 grant 语句都是即时生效的，那这么看应该就不需要执行 `flush privileges` 语句了呀。

答案也确实是这样的。

flush privileges 命令会清空 `acl_users` 数组，然后从 `mysql.user` 表中读取数据重新加载，重新构造一个 `acl_users` 数组。也就是说，以数据表中的数据为准，会将全局权限内存数组重新加载一遍。

同样地，对于 db 权限、表权限和列权限，MySQL 也做了这样的处理。

也就是说，如果内存的权限数据和磁盘数据表相同的话，不需要执行 flush privileges。而如果我们都是用 grant/revoke 语句来执行的话，内存和数据表本来就是保持同步更新的。

**因此，正常情况下，grant 命令之后，没有必要跟着执行 flush privileges 命令。**

### flush privileges 使用场景

那么，flush privileges 是在什么时候使用呢？显然，当数据表中的权限数据跟内存中的权限数据不一致的时候，flush privileges 语句可以用来重建内存数据，达到一致状态。

这种不一致往往是由不规范的操作导致的，比如直接用 DML 语句操作系统权限表。我们来看一下下面这个场景：

|      | client A                                                     | client B                                          |
| ---- | ------------------------------------------------------------ | ------------------------------------------------- |
| T1   | connect(root, root);<br />create user 'ua'@'%' identified by 'pa'; |                                                   |
| T2   |                                                              | connect(ua,pa);<br />(connect ok)<br />disconnect |
| T3   | delete from mysql.user where user = 'ua';                    |                                                   |
| T4   |                                                              | connect(ua, pa)<br />(connect ok)<br />disconnect |
| T5   | flush privileges;                                            |                                                   |
| T6   |                                                              | connect(ua, pa)<br />(Access Denied)              |

可以看到，T3 时刻虽然已经用 delete 语句删除了用户 ua，但是在 T4 时刻，仍然可以用 ua 连接成功。原因就是，这时候内存中 `acl_users` 数组中还有这个用户，因此系统判断时认为用户还正常存在。

在 T5 时刻执行过 flush 命令后，内存更新，T6 时刻再要用 ua 来登录的话，就会报错“无法访问”了。

直接操作系统表是不规范的操作，这个不一致状态也会导致一些更“诡异”的现象发生。比如，前面这个通过 delete 语句删除用户的例子，就会出现下面的情况：

### 小结

今天这篇文章，我和你介绍了 MySQL 用户权限在数据表和内存中的存在形式，以及 grant 和 revoke 命令的执行逻辑。

grant 语句会同时修改数据表和内存，判断权限的时候使用的是内存数据。因此，规范地使用 grant 和 revoke 语句，是不需要随后加上 flush privileges 语句的。

flush privileges 语句本身会用数据表的数据重建一份内存权限数据，所以在权限数据可能存在不一致的情况下再使用。而这种不一致往往是由于直接用 DML 语句操作系统权限表导致的，所以我们尽量不要使用这类语句。

另外，在使用 grant 语句赋权时，你可能还会看到这样的写法：

```sql
grant super on *.* to 'ua'@'%' identified by 'pa';
```

这条命令加了 identified by ‘密码’， 语句的逻辑里面除了赋权外，还包含了：

1. 如果用户 `’ua’@’%'`不存在，就创建这个用户，密码是 pa；
2. 如果用户 ua 已经存在，就将密码修改成 pa。

这也是一种不建议的写法，因为这种写法很容易就会不慎把密码给改了。

“grant 之后随手加 flush privileges”，我自己是这么使用了两三年之后，在看代码的时候才发现其实并不需要这样做，那已经是 2011 年的事情了。



