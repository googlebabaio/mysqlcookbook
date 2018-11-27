
<!-- toc --> 

* * * * *

# 一、profiler简介
MySQL 的 Query Profiler 是一个使用非常方便的 Query 诊断分析工具，通过该工具可以获取一条Query 在整个执行过程中多种资源的消耗情况，如 CPU，IO，IPC，SWAP 等，以及发生的 PAGE FAULTS，CONTEXT SWITCHE 等等，同时还能得到该 Query 执行过程中 MySQL 所调用的各个函数在源文件中的位置。

# 二、使用
## 1.查看是否已经启用profile，默认是关闭的
```
mysql> select @@profiling;
+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+
1 row in set, 1 warning (0.00 sec)

```

## 2.启用profiling
变量profiling是session变量，每次使用都需要重新启用。
 ```
 mysql> set profiling = 1;  
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> select @@profiling; 
+-------------+
| @@profiling |
+-------------+
|           1 |
+-------------+
1 row in set, 1 warning (0.00 sec)
 ```
 
## 3.使用示例
### 3.1 执行一个查询语句
为避免之前已经把 SQL 存放在 QCACHE 中， 建议在执行 SQL 时， 强制 SELECT 语句不进行 QCACHE 检测。
```
mysql> select sql_no_cache  count(*) from user;
+----------+
| count(*) |
+----------+
|        5 |
+----------+
1 row in set, 1 warning (0.00 sec)
```

### 3.2 使用show profile查询最近一条语句的执行信息
```
mysql> show profile;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000109 |
| checking permissions | 0.000009 |
| Opening tables       | 0.000025 |
| init                 | 0.000020 |
| System lock          | 0.000022 |
| optimizing           | 0.000009 |
| executing            | 0.000010 |
| end                  | 0.000004 |
| query end            | 0.000006 |
| closing tables       | 0.000011 |
| freeing items        | 0.000014 |
| cleaning up          | 0.000013 |
+----------------------+----------+
12 rows in set, 1 warning (0.01 sec)
```

### 3.3 使用show profiles查看在服务器上执行语句的列表。(查询id，花费时间，语句) 
```
mysql> show profiles;
+----------+------------+-----------------------------------------+
| Query_ID | Duration   | Query                                   |
+----------+------------+-----------------------------------------+
|        1 | 0.00007625 | elect @@profiling                       |
|        2 | 0.00015775 | select @@profiling                      |
|        3 | 0.00013875 | SELECT DATABASE()                       |
|        4 | 0.00014025 | SELECT DATABASE()                       |
|        5 | 0.00031725 | show databases                          |
|        6 | 0.00024500 | show tables                             |
|        7 | 0.00025050 | select sql_no_cache  count(*) from user |
+----------+------------+-----------------------------------------+
7 rows in set, 1 warning (0.00 sec)
```
### 3.4 使用show profile查询制定ID的执行信息。
这里分析ID为7的语句。（分析：`select sql_no_cache  count(*) from user;`)
```
mysql> show profile for query 7;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000109 |
| checking permissions | 0.000009 |
| Opening tables       | 0.000025 |
| init                 | 0.000020 |
| System lock          | 0.000022 |
| optimizing           | 0.000009 |
| executing            | 0.000010 |
| end                  | 0.000004 |
| query end            | 0.000006 |
| closing tables       | 0.000011 |
| freeing items        | 0.000014 |
| cleaning up          | 0.000013 |
+----------------------+----------+
12 rows in set, 1 warning (0.00 sec)
```
### 3.5 获取 CPU 和 Block IO 的消耗
```
mysql> show profile block io,cpu for query 7;
+----------------------+----------+----------+------------+--------------+---------------+
| Status               | Duration | CPU_user | CPU_system | Block_ops_in | Block_ops_out |
+----------------------+----------+----------+------------+--------------+---------------+
| starting             | 0.000109 | 0.000054 |   0.000046 |            0 |             0 |
| checking permissions | 0.000009 | 0.000005 |   0.000004 |            0 |             0 |
| Opening tables       | 0.000025 | 0.000013 |   0.000011 |            0 |             0 |
| init                 | 0.000020 | 0.000011 |   0.000009 |            0 |             0 |
| System lock          | 0.000022 | 0.000012 |   0.000010 |            0 |             0 |
| optimizing           | 0.000009 | 0.000005 |   0.000004 |            0 |             0 |
| executing            | 0.000010 | 0.000005 |   0.000005 |            0 |             0 |
| end                  | 0.000004 | 0.000002 |   0.000002 |            0 |             0 |
| query end            | 0.000006 | 0.000003 |   0.000002 |            0 |             0 |
| closing tables       | 0.000011 | 0.000006 |   0.000005 |            0 |             0 |
| freeing items        | 0.000014 | 0.000007 |   0.000006 |            0 |             0 |
| cleaning up          | 0.000013 | 0.000007 |   0.000006 |            0 |             0 |
+----------------------+----------+----------+------------+--------------+---------------+
12 rows in set, 1 warning (0.00 sec)
```

### 3.6 获取其他信息
通过执行`SHOW PROFILE *** FOR QUERY n` 来获取。
参考：http://dev.mysql.com/doc/refman/5.7/en/show-profile.html。
```
mysql> show profile all for query 7;
mysql> show profile cpu,block io,memory,swaps,context switches,source for query 6;
```