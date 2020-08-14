# 安装mgr
## 初始化环境
yum -y install libaio*


wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.21-linux-glibc2.12-x86_64.tar.xz

xz -d mysql-8.0.21-linux-glibc2.12-x86_64.tar.xz
tar -xvf mysql-8.0.21-linux-glibc2.12-x86_64.tar -C /usr/local
ln -s /usr/local/mysql-8.0.21-linux-glibc2.12-x86_64/ /usr/local/mysql
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
mkdir -p /u01/mysql_data/data/
chown mysql:mysql -R /u01/mysql_data/data
chmod 750 -R /u01//mysql_data/data


## 初始化MySQL
/usr/local/mysql/bin/mysqld --initialize-insecure --basedir=/usr/local/mysql --datadir=/u01/mysql_data/data --user=mysql --lower-case-table-names=1


## my.cnf文件
mv /etc/my.cnf /tmp/my.cnf.bak

vi /etc/my.cnf

```
[client]
#user = root
#password = xxxxx

[mysqld]
# basic settings #
user = mysql
port = 3306
socket = /tmp/mysql.sock
basedir = /usr/local/mysql
datadir =  /u01/mysql_data/data
bind-address = 0.0.0.0
max_connections = 5000
max_connect_errors = 100000
log_error = error.log
pid-file = mysql.pid
server-id = 3306137
log_bin = binlog
binlog_format = row
sync_binlog = 1
log_timestamps=SYSTEM
interactive_timeout = 28800
wait_timeout = 28800
default_authentication_plugin=mysql_native_password
transaction_isolation = READ-COMMITTED
binlog_rows_query_log_events = 1
lower-case-table-names=1 

# group repl settings #
gtid_mode=ON
enforce_gtid_consistency=ON
binlog_checksum=NONE

plugin_load_add='group_replication.so'
group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
group_replication_start_on_boot=off
group_replication_local_address= "172.24.18.148:33061"
group_replication_group_seeds= "172.24.18.150:33061,172.24.18.147:33061,172.24.18.148:33061"
group_replication_bootstrap_group=off
group_replication_unreachable_majority_timeout=60
group_replication_single_primary_mode = 1
#group_replication_enforce_update_everywhere_checks = 1


# innodb settings #
innodb_buffer_pool_size = 12G
innodb_buffer_pool_instances = 2
innodb_flush_log_at_trx_commit  = 1
```

> group_replication_group_seeds= "172.24.18.140:33061,172.24.18.141:33062,172.24.18.142:33063"
33061 33062 33063 不是指的端口


## 启动MySQL

/usr/local/mysql/bin/mysqld_safe  --defaults-file=/etc/my.cnf  --user=mysql &

或者：
cd /usr/local/mysql
cp support-files/mysql.server /etc/init.d/mysql
chkconfig --add mysql


systemctl start mysql

more /u01/mysql_data/data/error.log

## 配置bin
echo "export PATH=$PATH:/usr/local/mysql/bin" >> /etc/profile

source /etc/profile


## 配置Mgr
主库：
mysql -uroot
CREATE USER repl@'%' IDENTIFIED BY 'star_2020';
GRANT REPLICATION SLAVE ON *.* TO repl@'%';
FLUSH PRIVILEGES;

CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='star_2020' FOR CHANNEL 'group_replication_recovery';

INSTALL PLUGIN group_replication SONAME 'group_replication.so';

首次启动组的过程称为引导，只需要在一台服务器上执行一次，此库即为主库：

SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;


#查询集群状态
SELECT * FROM performance_schema.replication_group_members;

 

从库：
CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='star_2020' FOR CHANNEL 'group_replication_recovery';


START GROUP_REPLICATION;


## 修改密码
alter user 'root'@'%' identified by 'star_2020';
flush privileges;

create user 'root'@'%' IDENTIFIED WITH mysql_native_password by 'star_2020';
grant ALL ON *.* TO 'root'@'%' ;
flush privileges;

# 参考
https://www.cnblogs.com/9527l/p/12435675.html



---

# 安装配置proxysql
在任意一台机器，或者新增加一台机器做这个事情。

## 安装proxysql
cat <<EOF | tee /etc/yum.repos.d/proxysql.repo
[proxysql_repo]
name= ProxySQL YUM repository
baseurl=https://repo.proxysql.com/ProxySQL/proxysql-2.0.x/centos/7
gpgcheck=1
gpgkey=https://repo.proxysql.com/ProxySQL/repo_pub_key
EOF


yum install proxysql 


## 启动
systemctl start proxysql

netstat -nltp

## 配置
mysql -uadmin -padmin -h 127.0.0.1 -P6032 --prompt='Admin> '

insert into mysql_servers(hostgroup_id,hostname,port) values (10,'172.24.18.150',3306);
insert into mysql_servers(hostgroup_id,hostname,port) values (10,'172.24.18.148',3306);
insert into mysql_servers(hostgroup_id,hostname,port) values (10,'172.24.18.147',3306);

load mysql servers to runtime;
save mysql servers to disk;


## 在MySQL上配置
mysql -uroot -pstar_2020

CREATE USER 'monitor'@'%' IDENTIFIED BY "star_proxy_2020";
CREATE USER 'proxysql'@'%' IDENTIFIED BY "star_proxy_2020";
GRANT ALL PRIVILEGES ON *.* TO 'monitor'@'%' ;
GRANT ALL PRIVILEGES ON *.* TO 'proxysql'@'%' ;
flush privileges;

## 在Proxysql上设置监控账号与程序账号

set mysql-monitor_username='monitor';
set mysql-monitor_password='star_proxy_2020';

insert into mysql_users(username,password,active,default_hostgroup,transaction_persistent) values('proxysql','star_proxy_2020',1,10,1);

## 在MySQL上配置存储过程
mysql -uroot -pstar_2020

```
USE sys;
 
DELIMITER $$
 
CREATE FUNCTION my_id() RETURNS TEXT(36) DETERMINISTIC NO SQL RETURN (SELECT @@global.server_uuid as my_id);$$
 
CREATE FUNCTION gr_member_in_primary_partition()
    RETURNS VARCHAR(3)
    DETERMINISTIC
    BEGIN
      RETURN (SELECT IF( MEMBER_STATE='ONLINE' AND ((SELECT COUNT(*) FROM
    performance_schema.replication_group_members WHERE MEMBER_STATE NOT IN ('ONLINE', 'RECOVERING')) >=
    ((SELECT COUNT(*) FROM performance_schema.replication_group_members)/2) = 0),
    'YES', 'NO' ) FROM performance_schema.replication_group_members JOIN
    performance_schema.replication_group_member_stats USING(member_id) where member_id=my_id());
END$$
 
CREATE VIEW gr_member_routing_candidate_status AS SELECT
sys.gr_member_in_primary_partition() as viable_candidate,
IF( (SELECT (SELECT GROUP_CONCAT(variable_value) FROM
performance_schema.global_variables WHERE variable_name IN ('read_only',
'super_read_only')) != 'OFF,OFF'), 'YES', 'NO') as read_only,
Count_Transactions_Remote_In_Applier_Queue as transactions_behind, Count_Transactions_in_queue as 'transactions_to_cert' 
from performance_schema.replication_group_member_stats where member_id=my_id();$$
```

## 设置读写组
主负责写、从负责读，当MGR主库切换后，代理自动识别主从。
ProxySQL代理每一个后端MGR集群时，都必须为这个MGR定义写组10、备写组20、读组30、离线组40，

mysql -u admin -p admin -h 127.0.0.1 -P6032 --prompt='Admin> '

insert into mysql_group_replication_hostgroups (writer_hostgroup,backup_writer_hostgroup,reader_hostgroup, offline_hostgroup,active,max_writers,writer_is_also_reader,max_transactions_behind) values (10,20,30,40,1,1,0,400);

注意：max_transactions_behind 是设置延迟大小，可以给大点,建议自己去开个并行复制。

load mysql servers to runtime;
save mysql servers to disk;
load mysql users to runtime;
save mysql users to disk;
load mysql variables to runtime;
save mysql variables to disk;

## 查看ProxySQL对MGR监控状态
select hostname,port,viable_candidate,read_only,transactions_behind,error 
from mysql_server_group_replication_log
order by time_start_us desc limit 6;

## 配置读写分离规则
mysql -u admin -p admin -h 127.0.0.1 -P6032 --prompt='Admin> '

insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply)
VALUES (1,1,'^SELECT.*FOR UPDATE$',10,1),
(2,1,'^SELECT',30,1);

load mysql query rules to runtime;
save mysql query rules to disk;
load mysql servers to runtime;
save mysql servers to disk;
load mysql users to runtime;
save mysql users to disk;
load mysql variables to runtime;
save mysql variables to disk;

# 测试
mysql -uproxysql -pstar_proxy_2020 -h127.0.0.1 -P6033


create database test;
use test

create table t(id int primary key,name varchar(20),age int);

insert into t values(1,"hehe",29);

select * from test.t;

 

#查看路由规则

mysql -uadmin -padmin -h127.0.0.1 -P6032 --prompt='Admin> '
select hostgroup,digest_text from stats_mysql_query_digest order by digest_text desc limit 5;


https://www.cnblogs.com/yhq1314/p/11315379.html


---
# 基准测试

sysbench --db-driver=mysql  --mysql-host=172.24.18.150 --mysql-port=3306 --mysql-user=proxysql --mysql-password=star_proxy_2020 --mysql-db=sbtest   --table-size=10000 --tables=100 --enents=0 --time=150 --threads=32 --report_interval=5 --percentile=95 oltp_read_write prepare

sysbench --db-driver=mysql  --mysql-host=172.24.18.152 --mysql-port=3306 --mysql-user=proxysql --mysql-password=star_proxy_2020 --mysql-db=sbtest   --table-size=10000 --tables=100  --time=150 --threads=32 --report_interval=5 --percentile=95 oltp_read_write prepare





sysbench /usr/local/share/sysbench/tests/include/oltp_legacy/oltp.lua --mysql-host=172.24.18.152 --mysql-port=3306 --mysql-user=proxysql --mysql-password=star_proxy_2020 --mysql-db=sbtest --oltp-test-mode=complex --oltp-tables-count=100 --oltp-table-size=9999 --threads=16 --time=120 --report-interval=10 run 


测试只读
```
sysbench --test=/usr/share/doc/sysbench/tests/db/oltp.lua --mysql-user=xxxx --mysql-password=xxxxx --mysql-port=xxxxx --mysql-host=xxxxx --mysql-db=xxxx  --oltp-tables-count=100 --num-threads=32 run --oltp-skip-trx=on --oltp-read-only=on
```