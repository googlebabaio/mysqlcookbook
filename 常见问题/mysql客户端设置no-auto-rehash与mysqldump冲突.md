我们的数据库有许多表,列有很多列.命令行mysql客户端连接需要很长时间,除非我传递它-A.我宁愿不必每次都把它放进去,所以我尝试添加my.cnf选项no-auto-rehash.

这很好用,直到我必须使用mysqldump：

mysqldump：未知选项’–no-auto-rehash’

显然mysqldump使用my.cnf的[client]部分中的选项,即使有一个单独的[mysqldump]部分.有没有办法使用no-auto-rehash并仍然有一个功能mysqldump？有没有[没有真正的mysql-client]部分？

谢谢.

在mysql论坛上提出同样的问题而没有回复：

http://forums.mysql.com/read.php?35,583760

最佳答案
在[mysql]部分中放入no-auto-rehash选项,而不是[client]

```
[mysql]
no-auto-rehash
```
