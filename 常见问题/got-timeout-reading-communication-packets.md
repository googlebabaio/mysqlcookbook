https://dev.mysql.com/doc/refman/5.5/en/communication-errors.html

可能导致Got Timeout reading communication packets错误的原因有如下几个：

A client attempts to access a database but has no privileges for it.（没有权限）

A client uses an incorrect password.(密码错误)

A connection packet does not contain the right information.(连接没有包含正确信息)

It takes more than connect_timeout seconds to obtain a connect packet. (获取连接信息起过connect_timeout的时长)

The client program did not call mysql_close() before exiting.(客户端没有调用mysql_close()函数)

The client had been sleeping more than wait_timeout or interactive_timeout seconds without issuing any requests to the server. (客户端的空连接时间过长，超过了wait_timeout和interactive_timeout的时间)

The client program ended abruptly in the middle of a data transfer.(数据传输过程中终结)



https://blog.csdn.net/qq_43061705/article/details/93966592
