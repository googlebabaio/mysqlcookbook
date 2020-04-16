修改back_log参数值:由默认的50修改为500.（每个连接256kb, 占用：125M）
```MySQL
back_log=500
```


查看mysql 当前系统默认back_log值，命令：
```MySQL
show variables like 'back_log';
```

back_log值指出在MySQL暂时停止回答新请求之前的短时间内多少个请求可以被存在堆栈中。也就是说，如果MySql的连接数达到max_connections时，新来的请求将会被存在堆栈中，以等待某一连接释放资源，该堆栈的数量即back_log，如果等待连接的数量超过back_log，将不被授予连接资源。将会报：

```MySQL
unauthenticated user | xxx.xxx.xxx.xxx | NULL | Connect | NULL | login | NULL 的待连接进程时.
```

back_log值不能超过TCP/IP连接的侦听队列的大小。若超过则无效，查看当前系统的TCP/IP连接的侦听队列的大小命令：cat /proc/sys/net/ipv4/tcp_max_syn_backlog，目前系统为1024。对于Linux系统推荐设置为大于512的整数。

修改系统内核参数，可以编辑/etc/sysctl.conf去调整它。如：net.ipv4.tcp_max_syn_backlog = 2048，改完后执行sysctl -p 让修改立即生效。
