## 准备

两台机器，挂载盘用ext4格式化！

两台机器均有两个挂载盘：
* /data -- SSD，存储数据和bin log
* /backup -- 高速磁盘，做备份

## 安装

### 下载并安装MySQL官方的 Yum Repository

```shell
wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
yum -y install mysql57-community-release-el7-10.noarch.rpm
yum -y install mysql-community-server
```

### 通用设置

#### 修改配置文件

vim /etc/my.cnf，在[mysqld]节点下首先修改数据文件存放位置为我们刚刚挂载的SSD

```
datadir=/data/mysql
```

保存关闭

#### 初始化

首先启动MySQL：

```
systemctl start mysqld.service
```

设置自启动：

```
systemctl enable mysqld.service
```

由于第一次运行，MySQL会生成随机密码。找回：

```
grep "password" /var/log/mysqld.log
```

进入MySQL，设置root密码：

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'PASS';
```

如果设置的root密码太简单，可能提示'ERROR 1819'，需要修改validate_password_policy

```
set global validate_password_policy=0;
set global validate_password_length=10;
```

### 安装收尾

由于安装了Yum Repository，以后每次yum操作都会自动更新，需要把这个卸载掉：

```
yum -y remove mysql57-community-release-el7-10.noarch
```

但是如果需要mysql-lib或者mysql-devel这些类库，需要在卸载repository之前安装好。同理，如果卸载之后再安装这些类库，会提示找不到mariadb相应的版本，需要重新安装这个repository。

## 主从备份设置

首先停掉主从机的mysql：

```
systemctl stop mysqld.service
```

### 配置文件

放到文档最后面，有几个注意事项：
* binlog-do-db和replicate-do-db，如果需要支持多个，需要配置多行，而不要逗号分割

### master初始化及导出数据

启主的mysql：

```
systemctl start mysqld.service
```

加只读锁（如果没上线可以跳过这步）：

```sql
FLUSH TABLES WITH READ LOCK;
```

导出数据，--databases后的参数为需要导出的所有库，后面参数包含了触发器和存储过程：

```
mysqldump --databases db1 db2 db3 --single-transaction --triggers --routines --events --host=MASTER_IP --port=3306 --user=root --password=PASS > ~/all_db_backup.sql
```

重置master的bin log：
```sql
reset master;
```

释放master的只读锁（如果刚刚没加锁，跳过）：
```sql
unlock tables;
```

授权slave：
```sql
GRANT REPLICATION SLAVE ON *.* TO slave@SLAVE_IP IDENTIFIED BY 'PASS';
flush privileges;
```

复制导出文件到slave
```
scp -r all_db_backup.sql root@SLAVE_IP:~
```

### slave初始化及导入数据

启从的mysql：

```
systemctl start mysqld.service
```

导入数据：

```sql
source ~/all_db_backup.sql
```

修改存储过程的definer（因为导出的有些存储过程的host不正确）：

```sql
show procedure status where db="QAChannel_Business";
update mysql.proc set definer='root@localhost' where db = 'QAChannel_Business';
```

执行CHANGE MASTER TO

```sql
CHANGE MASTER TO MASTER_HOST='MASTER_IP',MASTER_USER='slave',MASTER_PASSWORD='PASS',MASTER_AUTO_POSITION=1;
```

启动slave：

```sql
start slave;
```

观察slave状态：

```sql
show slave status\G;
```

只有Slave_IO_Running和Slave_SQL_Running均为yes时，才表示启动正常，否则需要解决问题。问题在Last_Error描述，后面会列一些已知问题。

这样，Master-Slave就算配置好了

## 一致性保证与同步

生产环境，为了保证主从库的一致性，可以借助第三方工具Percona-Toolkit进行验证，我们主要用到两个：
* pt-table-checksum -- 用来检查主从库的一致性
* pt-table-sync -- 用来同步差异数据

### PT安装

安装perl相关的依赖：

```
yum install perl-DBD-MySQL  perl-IO-Socket-SSL perl-Digest-MD5 perl-TermReadKey
```

下载pt并安装：

```
wget https://www.percona.com/downloads/percona-toolkit/3.0.7/binary/redhat/7/x86_64/percona-toolkit-3.0.7-1.el7.x86_64.rpm
rpm -ivh percona-toolkit-3.0.7-1.el7.x86_64.rpm
```

建表，保存checksum计算结果：

```sql
CREATE DATABASE IF NOT EXISTS percona;
CREATE TABLE IF NOT EXISTS percona.checksums (
    db CHAR(64) NOT NULL,
    tbl CHAR(64) NOT NULL,
    chunk INT NOT NULL,
    chunk_time FLOAT NULL,
    chunk_index VARCHAR(200) NULL,
    lower_boundary TEXT NULL,
    upper_boundary TEXT NULL,
    this_crc CHAR(40) NOT NULL,
    this_cnt INT NOT NULL,
    master_crc CHAR(40) NULL,
    master_cnt INT NULL,
    ts TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (db,tbl,chunk),
    INDEX ts_db_tbl(ts,db,tbl)
) ENGINE=InnoDB;
```

### 检查数据一致性

```
pt-table-checksum --nocheck-binlog-format --replicate=percona.checksums --no-check-replication-filters --set-vars innodb_lock_wait_timeout=120 -uslave -p1qaz2wsx -h10.27.71.12 -P3306 --databases=short_urls
```

几个重要的参数：
* --nocheck-binlog-format：不检测日志格式。因为我们的binlog-format是ROW，而pt-table-checksum会在 Master和Slave 上设置binlog_format=STATEMENT
* --replicate：指定checksum计算结果存到哪个库表里，就是我们上面新建的表。
* --no-check-replication-filters：不需要检查Master配置里是否指定了Filter。因为我们配置了，并且默认检查，不加这个参数会报错。

输出结果如下：
```
Starting checksum ...
            TS ERRORS  DIFFS     ROWS  CHUNKS SKIPPED    TIME TABLE
03-13T10:27:19      0      1        3       1       0   0.276 short_urls.new_table
03-13T10:27:19      0      0       27       1       0   0.277 short_urls.short_urls
```

如果DIFFS不为0，则表示有数据主从不一致，需要同步。

### 同步：

```
pt-table-sync --execute --replicate=percona.checksums --charset=utf8 --databases=short_urls --no-check-slave u=slave,p=1qaz2wsx,h=10.27.71.12,P=3306 --print
```

执行后会打印同步结果，例如：
```
REPLACE INTO `short_urls`.`new_table` ......
```

## 主，挂了v(^_^)v

目前，如果master挂了，无法实现自动切换这样的高可用，需要手动将slave切换成master

首先停掉所有连接数据库的服务，将服务连接数据库的配置换成新的master（当然可以用slave来代替）。

然后确保每个小奴的relay log已经处理了所有statements：

```sql
STOP SLAVE IO_THREAD;
SHOW PROCESSLIST;
```

看到'Slave has read all relay log; waiting for more updates'，可以继续设置。

### 新master

如果用slave作为新的master：

```sql
stop slave;
```

然后同样的reset master，给小奴们授权：

```sql
reset master;
GRANT REPLICATION SLAVE ON *.* TO slave@NEW_SLAVE_IP IDENTIFIED BY 'PASS';
flush privileges;
```

这样master就上线了。

### 新slave

slave的设置参照上面即可，如果需要检查数据一致性，也要创建percona.checksums表。

测试时遇到过的问题，也可能是实际场景：

```
假设老master在挂的一瞬间，有新数据被写入，但是尚未同步到老slave，然后老slave被promoted成master，并且数据已
经开始写入，这时如果将老master修复并用作slave，同步时可能出现binlog不一致的错误（毕竟两个库都有新数据写入）。
```

这种情况目前没想到好的解决办法，只能丢弃掉老master挂那一瞬间写入的数据，然后再当作slave启动。

## 坑和填坑

#### slave启动时，因为同步错误而导致启动失败

解决方法：看show slave status\G;输出Last_Error内提示的哪个bin-log的哪个transaction出得问题，然后执行

```sql
STOP SLAVE;
SET @@SESSION.GTID_NEXT = 'transaction_id'; # Last_Error提示的事务ID
BEGIN; COMMIT;
SET @@SESSION.GTID_NEXT = AUTOMATIC;
START SLAVE;
```

之后检查数据一致性，如果有差异，同步。

#### 可能导致同步失败的原因

从目前看，所出现的不同步全是因为手欠（因为测试）造成的：
* bin-log丢失或改路径；
* 直接向从库插入数据，并导致同步时master数据因为主键冲突无法写入

#### 增加一个slave，启动的时候报错

[ERROR] Slave SQL: Slave failed to initialize relay log info structure from the repository, Error_code: 1872

解决方法：在slave上执行

#### mysqldump报错

[ERROR] mysqldump: Couldn't execute 'show events': Cannot proceed because system tables used by Event Scheduler were found damaged at server start (1577)

解决方法：mysql_upgrade -u root -h localhost -p --verbose --force

#### 执行mysql_upgrade报错

[ERROR] The mysql.session exists but is not correctly configured. The mysql.session needs SELECT privileges in the performance_schema database and the mysql.db table and also SUPER privileges

解决方法：往错误提示的两张表里各插入一条记录：

```sql
INSERT IGNORE INTO mysql.tables_priv VALUES ('localhost', 'mysql','mysql.session', 'user', 'root@localhost', CURRENT_TIMESTAMP,'Select', '');

insert into db (Host, Db, User, Select_priv) values ('localhost', 'performance_schema', 'mysql.session', 'Y');
```

如果执行依旧报错（大概是报找不到表，或者表非法之类的），参考下文"change master报错"的解决办法。

#### change master报错

[ERROR] Slave is not configured or failed to initialize properly. 

另外由于数据库版本升级（5.x -> 5.7），出现过许多问题（例如执行上面的mysql_upgrade报错），都可以尝试这样解决

解决方法：
* 首先drop掉如下表：

```sql
use mysql;
drop table mysql.innodb_index_stats;
drop table mysql.innodb_table_stats;
drop table mysql.slave_master_info;
drop table mysql.slave_relay_log_info;
drop table mysql.slave_worker_info;
```

* 然后执行

```sql
source /usr/share/mysql/mysql_system_tables.sql;
```

之后重启mysql，然后再change master

```sql
reset slave
```

