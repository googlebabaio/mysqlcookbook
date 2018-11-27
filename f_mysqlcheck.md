
<!-- toc --> 

* * * * *

## 使用mysqlcheck来检查和修复, 优化表

### 检查表
#### 检查特定的表
```
mysqlcheck -c employee dept -uroot -p
employee 是库名，dept是表名，还需要输入用户名和密码
```

#### 检查一个库中的所有表
```
mysqlcheck -c employee dept -uroot -p
```

#### 检查某几个库
```
 mysqlcheck -c --databases employess mysql test -uroot -p
```

#### 检查所有库中的所有表
```
mysqlcheck -c --all-databases -uroot -p
```

### 使用 mysqlcheck 分析表
```
mysqlcheck -a employee dept  -uroot -p
```

### 使用 mysqlcheck 优化表
```
mysqlcheck -o employee dept  -uroot -p
```

注意：
OPTIMIZE TABLE运行过程中，MySQL会锁定表。
innodb的引擎应该使用其他方式来完成优化，如：
```
//碎片整理
alter table table_name engine=innodb; 

//收集表的统计信息
analyze table table_name；
```

### 使用 mysqlcheck 修复表
```
mysqlcheck -r employee dept  -uroot -p
```

### 常用选项
```
A, –all-databases 表示所有库
-a, –analyze 分析表
-o, –optimize 优化表
-r, –repair 修复表错误
-c, –check 检查表是否出错
–auto-repair 自动修复损坏的表
-B, –databases 选择多个库
-1, –all-in-1 Use one query per database with tables listed in a comma separated way
-C, –check-only-changed 检查表最后一次检查之后的变动
-g, –check-upgrade Check for version dependent changes in the tables
-F, –fast Check tables that are not closed properly
–fix-db-names Fix DB names
–fix-table-names Fix table names
-f, –force Continue even when there is an error
-e, –extended Perform extended check on a table. This will take a long time to execute.
-m, –medium-check Faster than extended check option, but does most checks
-q, –quick Faster than medium check option
```