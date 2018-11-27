sample数据库类似Oracle安装时候提供的sample。将最新的dump文件从github上拉下来
```[root@nazeebo softdb]# git clone https://github.com/datacharmer/test_db.git
Cloning into 'test_db'...
remote: Counting objects: 102, done.
remote: Total 102 (delta 0), reused 0 (delta 0), pack-reused 102
Receiving objects: 100% (102/102), 68.81 MiB | 63.00 KiB/s, done.
Resolving deltas: 100% (54/54), done.
[root@nazeebo softdb]# 
[root@nazeebo softdb]# 
[root@nazeebo softdb]# 
[root@nazeebo softdb]# 
[root@nazeebo softdb]# pwd
/softdb
[root@nazeebo softdb]# ls
dbt2-0.37.50.15.tar.gz  sysbench-0.4.12.14  sysbench-0.4.12.14.tar.gz  test_db
[root@nazeebo softdb]# cd test_db/

[root@nazeebo test_db]# ll
total 168340
-rw-r--r-- 1 root root      964 Jun 22 14:24 Changelog
-rw-r--r-- 1 root root     7948 Jun 22 14:24 employees_partitioned_5.1.sql
-rw-r--r-- 1 root root     6276 Jun 22 14:24 employees_partitioned.sql
-rw-r--r-- 1 root root     4193 Jun 22 14:24 employees.sql
drwxr-xr-x 2 root root     4096 Jun 22 14:24 images
-rw-r--r-- 1 root root      250 Jun 22 14:24 load_departments.dump
-rw-r--r-- 1 root root 14159880 Jun 22 14:24 load_dept_emp.dump
-rw-r--r-- 1 root root     1090 Jun 22 14:24 load_dept_manager.dump
-rw-r--r-- 1 root root 17722832 Jun 22 14:24 load_employees.dump
-rw-r--r-- 1 root root 39806034 Jun 22 14:24 load_salaries1.dump
-rw-r--r-- 1 root root 39805981 Jun 22 14:24 load_salaries2.dump
-rw-r--r-- 1 root root 39080916 Jun 22 14:24 load_salaries3.dump
-rw-r--r-- 1 root root 21708736 Jun 22 14:24 load_titles.dump
-rw-r--r-- 1 root root     4568 Jun 22 14:24 objects.sql
-rw-r--r-- 1 root root     4009 Jun 22 14:24 README.md
drwxr-xr-x 2 root root     4096 Jun 22 14:24 sakila
-rw-r--r-- 1 root root      272 Jun 22 14:24 show_elapsed.sql
-rwxr-xr-x 1 root root     1800 Jun 22 14:24 sql_test.sh
-rw-r--r-- 1 root root     4878 Jun 22 14:24 test_employees_md5.sql
-rw-r--r-- 1 root root     4882 Jun 22 14:24 test_employees_sha.sql
```

导入
```
[root@nazeebo test_db]# mysql -uroot -p123456 < employees.sql 
mysql: [Warning] Using a password on the command line interface can be insecure.
INFO
CREATING DATABASE STRUCTURE
INFO
storage engine: InnoDB
INFO
LOADING departments
INFO
LOADING employees
INFO
LOADING dept_emp
INFO
LOADING dept_manager
INFO
LOADING titles
INFO
LOADING salaries
data_load_time_diff
00:01:09
```


验证：
```
[root@nazeebo test_db]# time mysql -uroot -p -t < test_employees_sha.sql 
Enter password: 
+----------------------+
| INFO                 |
+----------------------+
| TESTING INSTALLATION |
+----------------------+
+--------------+------------------+------------------------------------------+
| table_name   | expected_records | expected_crc                             |
+--------------+------------------+------------------------------------------+
| employees    |           300024 | 4d4aa689914d8fd41db7e45c2168e7dcb9697359 |
| departments  |                9 | 4b315afa0e35ca6649df897b958345bcb3d2b764 |
| dept_manager |               24 | 9687a7d6f93ca8847388a42a6d8d93982a841c6c |
| dept_emp     |           331603 | d95ab9fe07df0865f592574b3b33b9c741d9fd1b |
| titles       |           443308 | d12d5f746b88f07e69b9e36675b6067abb01b60e |
| salaries     |          2844047 | b5a1785c27d75e33a4173aaa22ccf41ebd7d4a9f |
+--------------+------------------+------------------------------------------+
+--------------+------------------+------------------------------------------+
| table_name   | found_records    | found_crc                                |
+--------------+------------------+------------------------------------------+
| employees    |           300024 | 4d4aa689914d8fd41db7e45c2168e7dcb9697359 |
| departments  |                9 | 4b315afa0e35ca6649df897b958345bcb3d2b764 |
| dept_manager |               24 | 9687a7d6f93ca8847388a42a6d8d93982a841c6c |
| dept_emp     |           331603 | d95ab9fe07df0865f592574b3b33b9c741d9fd1b |
| titles       |           443308 | d12d5f746b88f07e69b9e36675b6067abb01b60e |
| salaries     |          2844047 | b5a1785c27d75e33a4173aaa22ccf41ebd7d4a9f |
+--------------+------------------+------------------------------------------+
+--------------+---------------+-----------+
| table_name   | records_match | crc_match |
+--------------+---------------+-----------+
| employees    | OK            | ok        |
| departments  | OK            | ok        |
| dept_manager | OK            | ok        |
| dept_emp     | OK            | ok        |
| titles       | OK            | ok        |
| salaries     | OK            | ok        |
+--------------+---------------+-----------+
+------------------+
| computation_time |
+------------------+
| 00:00:15         |
+------------------+
+---------+--------+
| summary | result |
+---------+--------+
| CRC     | OK     |
| count   | OK     |
+---------+--------+

real	0m17.996s
user	0m0.004s
sys	0m0.004s
[root@nazeebo test_db]# time mysql -uroot -p -t < test_employees_md5.sql 
Enter password: 
+----------------------+
| INFO                 |
+----------------------+
| TESTING INSTALLATION |
+----------------------+
+--------------+------------------+----------------------------------+
| table_name   | expected_records | expected_crc                     |
+--------------+------------------+----------------------------------+
| employees    |           300024 | 4ec56ab5ba37218d187cf6ab09ce1aa1 |
| departments  |                9 | d1af5e170d2d1591d776d5638d71fc5f |
| dept_manager |               24 | 8720e2f0853ac9096b689c14664f847e |
| dept_emp     |           331603 | ccf6fe516f990bdaa49713fc478701b7 |
| titles       |           443308 | bfa016c472df68e70a03facafa1bc0a8 |
| salaries     |          2844047 | fd220654e95aea1b169624ffe3fca934 |
+--------------+------------------+----------------------------------+
+--------------+------------------+----------------------------------+
| table_name   | found_records    | found_crc                        |
+--------------+------------------+----------------------------------+
| employees    |           300024 | 4ec56ab5ba37218d187cf6ab09ce1aa1 |
| departments  |                9 | d1af5e170d2d1591d776d5638d71fc5f |
| dept_manager |               24 | 8720e2f0853ac9096b689c14664f847e |
| dept_emp     |           331603 | ccf6fe516f990bdaa49713fc478701b7 |
| titles       |           443308 | bfa016c472df68e70a03facafa1bc0a8 |
| salaries     |          2844047 | fd220654e95aea1b169624ffe3fca934 |
+--------------+------------------+----------------------------------+
+--------------+---------------+-----------+
| table_name   | records_match | crc_match |
+--------------+---------------+-----------+
| employees    | OK            | ok        |
| departments  | OK            | ok        |
| dept_manager | OK            | ok        |
| dept_emp     | OK            | ok        |
| titles       | OK            | ok        |
| salaries     | OK            | ok        |
+--------------+---------------+-----------+
+------------------+
| computation_time |
+------------------+
| 00:00:13         |
+------------------+
+---------+--------+
| summary | result |
+---------+--------+
| CRC     | OK     |
| count   | OK     |
+---------+--------+

real	0m16.595s
user	0m0.002s
sys	0m0.007s
```

验证2：
```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| employees          |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
6 rows in set (0.00 sec)

mysql> use employees
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------------+
| Tables_in_employees  |
+----------------------+
| current_dept_emp     |
| departments          |
| dept_emp             |
| dept_emp_latest_date |
| dept_manager         |
| employees            |
| salaries             |
| titles               |
+----------------------+
8 rows in set (0.00 sec)
```
相关的ER图
![](images/screenshot_1531376923320.png)