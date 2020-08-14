
# 初始化参数
1.binlog_checksum
```
禁用binlog_checksum
binlog_checksum = NONE
MGR不支持带checksum的binlog event。
```

2.master_info_repository和relay_log_info_repository
```
使用系统表来存储slave信息
master_info_repository = TABLE
relay_log_info_repository = TABLE
对应如下表:
mysql.slave_master_info
mysql.slave_relay_log_info
```

3.transaction_write_set_extraction
```
transaction_write_set_extraction = XXHASH64
Group Replication要求每个表必须要有主键，用来做冲突检测。
组内所有成员必须配置相同的哈希算法。
```

4.group_replication_group_name
```
group_replication_group_name = '5a421130-2674-11ea-bbce-00505639ee45'
一个UUID值，组内唯一，组内都是用这个值，用来标记组内所有成员上产生的binlog event。
```

5.group_replication_local_address
```
group_replication_local_address = 'db03:33061'
每个成员都有一个独立的TCP端口,成员之间通过这个端口进行通信。
```

6.group_replication_group_seeds
```
group_replication_group_seeds = 'db01:33061,db02:33061,db03:33061'
设置种子成员的地址
当加入一个组时,新成员首先需要和组内的成员进行通信来完成加入的步骤,因此需要知道至少一个当前组内成员的地址。
```


7.group_replication_bootstrap_group
```
只在组内的一个成上执行，用来初始化组,其它成员无需执行。
set global group_replication_bootstrap_group=ON;
start group_replication;
set global group_replication_bootstrap_group=OFF;
```


8.group_replication_start_on_boot
```
group_replication_start_on_boot = off
指示插件mysql启动的时候不要自动启动组复制
```

# 监控

Group Replication的状态信息被存储到了performance_schema中的几个表中:
- replication_group_members
- replication_group_member_stats
- replication_applier_status
- replication_connection_status

## replication_group_members:
存储着组内所有成员的基本信息，从任何一个成员上都能查询到这些基本信息。
- CHANNEL_NAME:Group Replication执行Binlog Event的通道,值为group_replication_applier。
- MEMBER_STATE:成员状态。
  - OFFLINE:组复制插件没启动,为OFFLINE状态。
  - RECOVERING:当组复制插件启动时,首先设置为此状态,开始复制加入前的数据。
  - ONLINE:recover完之后,设置为此状态,开始对外提供服务。
  - ERROR:当本地成员发生错误时,设置为此状态。
  - UNREACHABLE:网络故障或宕机,设置为此状态。

## replication_group_member_stats:
存储着本地成员的详细信息,每个成员上只能查询到自己的详细信息。
- CHANNEL_NAME:Group Replication执行Binlog Event的通道,值为group_replication_applier。
- VIEW_ID:组视图ID
- COUNT_TRANSACTIONS_IN_QUEUE:队列中等待做全局事务认证的事务数量。
- COUNT_TRANSACTIONS_CHECKED:做了全局事务认证的事务总数量。
- COUNT_CONFLICTS_DETECTED:全局事务认证时,有冲突的事务总数量。
- COUNT_TRANSACTIONS_ROWS_VALIDATING:冲突检测数据库的记录总行数。
- TRANSACTIONS_COMMITTED_ALL_MEMBERS:在所有成员上已经执行的事务的GTID集合。不是实时的,每隔一段时间更新一次。
- LAST_CONFLICT_FREE_TRANSACTION:最后一个没有冲突的事务的GTID。

## replication_connection_status
异步复制通道连接信息

## replication_applier_status
观察与组复制相关的通道和线程的状态。如果有许多不同的工作线程在应用事务，那么工作表也可以用来监视每个工作线程在做什么。

## group_replication_applier
此通道用于来自组的传入更改。这是应用直接来自该组的交易的渠道。

## group_replication_recovery
此通道用于与分布式恢复阶段相关的复制更改。

# 一个troubleshooting的例子
查看状态
```
SELECT * FROM performance_schema.replication_group_members;
```

去某个节点查看复制的状态
128:
```
select * from performance_schema.replication_group_member_stats \G ;
*************************** 1. row ***************************
                      CHANNEL_NAME: group_replication_applier
                           VIEW_ID: 15263657103992362:28
                         MEMBER_ID: b376058d-5762-11e7-baa0-000c29e6b568
       COUNT_TRANSACTIONS_IN_QUEUE: 0
        COUNT_TRANSACTIONS_CHECKED: 0
          COUNT_CONFLICTS_DETECTED: 0
COUNT_TRANSACTIONS_ROWS_VALIDATING: 0
TRANSACTIONS_COMMITTED_ALL_MEMBERS: 
    LAST_CONFLICT_FREE_TRANSACTION:
```

129:
```
root@localhost:qn 04:47:47>select * from performance_schema.replication_group_member_stats \G ;
*************************** 1. row ***************************
                      CHANNEL_NAME: group_replication_applier
                           VIEW_ID: 15263657103992362:28
                         MEMBER_ID: 574d6e9a-fc15-11e7-9fcf-000c29879d51
       COUNT_TRANSACTIONS_IN_QUEUE: 0
        COUNT_TRANSACTIONS_CHECKED: 1
          COUNT_CONFLICTS_DETECTED: 0
COUNT_TRANSACTIONS_ROWS_VALIDATING: 0
TRANSACTIONS_COMMITTED_ALL_MEMBERS: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1-151,
b376058d-5762-11e7-baa0-000c29e6b568:1-47678
    LAST_CONFLICT_FREE_TRANSACTION: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:142
1 row in set (0.00 sec)
```

130:
```
root@localhost:(none) 05:05:06>select * from performance_schema.replication_group_member_stats \G ;
*************************** 1. row ***************************
                      CHANNEL_NAME: group_replication_applier
                           VIEW_ID: 15263657103992362:33
                         MEMBER_ID: e87b5168-fc15-11e7-9fcf-000c29879d51
       COUNT_TRANSACTIONS_IN_QUEUE: 5
        COUNT_TRANSACTIONS_CHECKED: 1
          COUNT_CONFLICTS_DETECTED: 0
COUNT_TRANSACTIONS_ROWS_VALIDATING: 0
TRANSACTIONS_COMMITTED_ALL_MEMBERS: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1-151,
b376058d-5762-11e7-baa0-000c29e6b568:1-47678
    LAST_CONFLICT_FREE_TRANSACTION: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:142
```

查看128的GTID状态
```
root@localhost:(none) >show global variables like '%gtid%' ;
+---------------------------------------------------+---------+
| Variable_name                                     | Value   |
+---------------------------------------------------+---------+
| binlog_gtid_simple_recovery                       | ON      |
| enforce_gtid_consistency                          | ON      |
| group_replication_allow_local_disjoint_gtids_join | OFF     |
| group_replication_gtid_assignment_block_size      | 1000000 |
| gtid_executed                                     |         |
| gtid_executed_compression_period                  | 1000    |
| gtid_mode                                         | ON      |
| gtid_owned                                        |         |
| gtid_purged                                       |         |
| session_track_gtids                               | OFF     |
+---------------------------------------------------+---------+
```

129、130的：
```
root@localhost:qn 04:44:32>show global variables like '%gtid%' ;
+---------------------------------------------------+------------------------------------------------------------------------------------------+
| Variable_name                                     | Value                                                                                    |
+---------------------------------------------------+------------------------------------------------------------------------------------------+
| binlog_gtid_simple_recovery                       | ON                                                                                       |
| enforce_gtid_consistency                          | ON                                                                                       |
| group_replication_allow_local_disjoint_gtids_join | OFF                                                                                      |
| group_replication_gtid_assignment_block_size      | 1000000                                                                                  |
| gtid_executed                                     | aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1-149,
b376058d-5762-11e7-baa0-000c29e6b568:1-47678 |
| gtid_executed_compression_period                  | 1000                                                                                     |
| gtid_mode                                         | ON                                                                                       |
| gtid_owned                                        |                                                                                          |
| gtid_purged                                       | aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1-135,
b376058d-5762-11e7-baa0-000c29e6b568:1-47678 |
| session_track_gtids                               | OFF                                    
```

根据centos129与centos130的gtid_purged ,更改centos128的gtid_purged使其按gtid_purged开始复制:

```
root@localhost:(none) 04:53:03>set global gtid_purged='aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1-135,b376058d-5762-11e7-baa0-000c29e6b568:1-47678' ;
Query OK, 0 rows affected (0.24 sec)
``` 

```
root@localhost:(none) 04:53:17>show global variables like '%gtid%' ;
+---------------------------------------------------+------------------------------------------------------------------------------------------+
| Variable_name                                     | Value                                                                                    |
+---------------------------------------------------+------------------------------------------------------------------------------------------+
| binlog_gtid_simple_recovery                       | ON                                                                                       |
| enforce_gtid_consistency                          | ON                                                                                       |
| group_replication_allow_local_disjoint_gtids_join | OFF                                                                                      |
| group_replication_gtid_assignment_block_size      | 1000000                                                                                  |
| gtid_executed                                     | aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1-135,
b376058d-5762-11e7-baa0-000c29e6b568:1-47678 |
| gtid_executed_compression_period                  | 1000                                                                                     |
| gtid_mode                                         | ON                                                                                       |
| gtid_owned                                        |                                                                                          |
| gtid_purged                                       | aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1-135,
b376058d-5762-11e7-baa0-000c29e6b568:1-47678 |
| session_track_gtids                               | OFF                                                                                      |
+---------------------------------------------------+------------------------------------------------------------------------------------------+
```

开启128的复制
```sh
root@localhost:(none) 04:53:22>CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_pass' FOR CHANNEL 'group_replication_recovery';
Query OK, 0 rows affected, 2 warnings (0.02 sec)
 
root@localhost:(none) 04:54:17>show global variables like '%gtid%' ;
+---------------------------------------------------+------------------------------------------------------------------------------------------+
| Variable_name                                     | Value                                                                                    |
+---------------------------------------------------+------------------------------------------------------------------------------------------+
| binlog_gtid_simple_recovery                       | ON                                                                                       |
| enforce_gtid_consistency                          | ON                                                                                       |
| group_replication_allow_local_disjoint_gtids_join | OFF                                                                                      |
| group_replication_gtid_assignment_block_size      | 1000000                                                                                  |
| gtid_executed                                     | aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1-135,
b376058d-5762-11e7-baa0-000c29e6b568:1-47678 |
| gtid_executed_compression_period                  | 1000                                                                                     |
| gtid_mode                                         | ON                                                                                       |
| gtid_owned                                        |                                                                                          |
| gtid_purged                                       | aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1-135,
b376058d-5762-11e7-baa0-000c29e6b568:1-47678 |
| session_track_gtids                               | OFF                                                                                      |
+---------------------------------------------------+------------------------------------------------------------------------------------------+
10 rows in set (0.00 sec)
 
root@localhost:(none) 04:54:21>START GROUP_REPLICATION; 
Query OK, 0 rows affected (1.93 sec)
 
root@localhost:(none) 04:54:35>SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | 574d6e9a-fc15-11e7-9fcf-000c29879d51 | centos129   |        3306 | ONLINE       |
| group_replication_applier | b376058d-5762-11e7-baa0-000c29e6b568 | centos128   |        3306 | ONLINE       |
| group_replication_applier | e87b5168-fc15-11e7-9fcf-000c29879d51 | centos130   |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
3 rows in set (0.00 sec)
```


参考：
http://blog.sina.com.cn/s/blog_161dd36160102yltc.html
https://dev.mysql.com/doc/refman/5.7/en/group-replication-server-states.html
https://my.oschina.net/u/4582201/blog/4378125