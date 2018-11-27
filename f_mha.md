<!-- toc --> 

* * * * *

# 1. 环境介绍
ip地址 | 主机名 | 角色
--|---|--
192.168.0.175| nazeebo| master
192.168.0.176 | lowa | slave
192.168.0.200 | ai2018 | slave + manager

```
shell> cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
192.168.0.175 nazeebo
192.168.0.176 lowa
192.168.0.200 ai2018
```


# 2. 配置步骤
- 配置3台MySQL服务器1主2从(_基于gtid_)
- 在数据节点和管理节点安装相应的 mha 软件包
- 安装mha manager
- 配置ssh的互通
- 对从库进行一些设置:只读,relaylog清理
- 在管理节点配置 mha 的配置信息和脚本
- 添加虚拟IP

## 2.1.配置3台MySQL服务器1主2从(_基于gtid_)
### 2.1.1 在主库上创建复制用户和管理用户:
创建复制用户并授权
```
SQL> GRANT REPLICATION SLAVE ON *.* TO repluser@192.168.0.176 IDENTIFIED BY  'Oracle123';
SQL> GRANT REPLICATION SLAVE ON *.* TO repluser@192.168.0.200 IDENTIFIED BY  'Oracle123';
SQL> flush privileges;
```
创建管理用户并授权
```
SQL> grant all privileges on *.* to 'mhamon'@'192.168.0.200' identified  by 'Oracle123';
SQL> flush privileges;
```

### 2.1.2 在两台从库上配置主从复制
```
SQL> CHANGE MASTER TO MASTER_HOST='192.168.0.175',
MASTER_USER='repluser',
MASTER_PASSWORD='Oracle123',
MASTER_AUTO_POSITION=1;
```

## 2.2 安装相应的perl包

### 2.2.1 在每个节点安装mha4node
```
[root@lowa mha4mysql-0.57]# cd mha4mysql-node-0.57
[root@lowa mha4mysql-node-0.57]# ls
AUTHORS  bin  COPYING  debian  inc  lib  Makefile.PL  MANIFEST  META.yml  README  rpm  t
[root@lowa mha4mysql-node-0.57]# perl Makefile.PL
*** Module::AutoInstall version 1.06
*** Checking for Perl dependencies...
[Core Features]
- DBI        ...loaded. (1.609)
- DBD::mysql ...loaded. (4.013)
*** Module::AutoInstall configuration finished.
Checking if your kit is complete...
Looks good
Writing Makefile for mha4mysql::node
[root@lowa mha4mysql-node-0.57]# make && make install
...
这个安装很简单,基本不会有问题
```
>  Node工具包主要包括以下几个工具：
>
- save_binary_logs                保存和复制master的二进制日志
- apply_diff_relay_logs            识别差异的中继日志事件并将其差异的事件应用于其他的slave
- filter_mysqlbinlog               去除不必要的ROLLBACK事件（MHA已不再使用这个工具）
- purge_relay_logs                清除中继日志（不会阻塞SQL线程）
-

### 2.2.2 在管理节点安装mha4manager

安装相应的perl包
```
[root@ai2018 mha4mysql-manager-0.57]# perl -MDBD::mysql -e "print\"module installed\n\""
module installed
[root@ai2018 mha4mysql-manager-0.57]# perl -Config::Tiny -e "print\"module installed\n\""
Unknown Unicode option letter 'n'.
[root@ai2018 mha4mysql-manager-0.57]# ls
AUTHORS  bin  blib  COPYING  debian  inc  lib  Makefile  Makefile.PL  MANIFEST  META.yml  README  rpm  samples  t  tests
[root@ai2018 mha4mysql-manager-0.57]# perl Makefile.PL
*** Module::AutoInstall version 1.06
*** Checking for Perl dependencies...
[Core Features]
- DBI                   ...loaded. (1.609)
- DBD::mysql            ...loaded. (4.013)
- Time::HiRes           ...loaded. (1.9721)
- Config::Tiny          ...missing.
- Log::Dispatch         ...missing.
- Parallel::ForkManager ...loaded. (0.7.9)
- MHA::NodeConst        ...loaded. (0.57)
==> Auto-install the 2 mandatory module(s) from CPAN? [y] y
*** Dependencies will be installed the next time you type 'make'.
*** Module::AutoInstall configuration finished.
Warning: prerequisite Config::Tiny 0 not found.
Warning: prerequisite Log::Dispatch 0 not found.
Writing Makefile for mha4mysql::manager
```

这儿遇到了一些问题,manager需要依赖一些perl的包。先是采用yum的方式进行perl相关包的安装,结果不行,后来又把rpm下载下来安装,也不行..
```
yum install -y perl-DBD-MySQL.x86_64 \
                     perl-DBI.x86_64 perl-ExtUtils-CBuilder \
                     perl-ExtUtils-MakeMaker perl-CPAN.x86_64 \
                     perl-Mail-Sender perl-Log-Dispatch
```
最后耍了个赖,采用cpanm来进行安装.

安装mha4manager
```
[root@ai2018 mha4mysql-manager-0.57]# pwd
/softdb/mha4mysql-0.57/mha4mysql-manager-0.57
[root@ai2018 mha4mysql-manager-0.57]# ls
AUTHORS  bin  blib  COPYING  debian  inc  lib  Makefile  Makefile.PL  MANIFEST  META.yml  README  rpm  samples  t  tests
[root@ai2018 mha4mysql-manager-0.57]# perl Makefile.PL
*** Module::AutoInstall version 1.06
*** Checking for Perl dependencies...
[Core Features]
- DBI                   ...loaded. (1.609)
- DBD::mysql            ...loaded. (4.013)
- Time::HiRes           ...loaded. (1.9721)
- Config::Tiny          ...loaded. (2.23)
- Log::Dispatch         ...loaded. (2.67)
- Parallel::ForkManager ...loaded. (0.7.9)
- MHA::NodeConst        ...loaded. (0.57)
*** Module::AutoInstall configuration finished.
Generating a Unix-style Makefile
Writing Makefile for mha4mysql::manager
Writing MYMETA.yml and MYMETA.json
[root@ai2018 mha4mysql-manager-0.57]# make && make install
...
...
Appending installation info to /usr/lib64/perl5/perllocal.pod
```
> manager工具包主要包括以下几个工具：
>
- masterha_check_ssh              检查MHA的SSH配置状况
- masterha_check_repl             检查MySQL复制状况
- masterha_manger                  启动MHA
- masterha_check_status            检测当前MHA运行状态
- masterha_master_monitor          检测master是否宕机
- masterha_master_switch           控制故障转移（自动或者手动）
- masterha_conf_host               添加或删除配置的server信息

#### 2.3 配置ssh互通
采用ssh-copy-id是最方便的手段,如果OS上面没有这个命令,可以先用
`yum install openssl/openssl-client` 来进行安装

```
shell> ssh-keygen
shell> ssh-copy-id  -i  /root/.ssh/id_rsa.pub  "-p 10022  root@192.168.0.175"
shell> ssh-copy-id  -i  /root/.ssh/id_rsa.pub  "-p 10022  root@192.168.0.176"
shell> ssh-copy-id  -i  /root/.ssh/id_rsa.pub  "-p 10022  root@192.168.0.200"
```
```
或者修改全局的
vim /etc/ssh/ssh_config
Port 10022
```

## 2.4 对从库进行设置
### 2.4.1 设置只读
将两台slave服务器设置read_only
<kbd>从库对外提供读服务，只所以没有写进配置文件，是因为随时slave会提升为master </kbd>
```
在两个从库上执行
shell> mysql -uroot -p -e 'set global read_only=1'
```

### 2.4.2 relaylog的清理
mha在发生切换的过程中，从库的恢复过程会依赖于relay log的相关信息，所以这里要将relay log的自动清除设置为OFF。在默认情况下，从服务器上的中继日志会在SQL线程执行完毕后被自动删除。
```
在两个从库上执行
shell> mysql -uroot -p -e 'set global relay_log_purge=0'
```

## 2.5 mha管理节点的配置
先来个目录结构
```
[root@ai2018 mha]# tree /etc/mha
/etc/mha
├── app1
│   ├── app1.cnf
│   ├── app1.log #manager的日志
│   ├── app1.master_status.health #master的健康检查状态日志
│   └── app2.cnf #备份的app1.cnf
├── bin
│   └── mhaCli.sh  #mha启动脚本
└── script
    ├── master_ip_failover #自动切换的脚本
    └── sendEmail  #发送邮件的脚本,在这儿没有需求,所以没有配置

3 directories, 7 files
```
### 2.5.1 节点的相关脚本配置
app1.cnf
```
[server default]
manager_log=/etc/mha/app1/app1.log    # 设置manager的日志
manager_workdir=/etc/mha/app1/      # 设置manager的工作目录
master_binlog_dir=/u01/mysql/mysql_data     # 设置master 保存binlog的位置，以便MHA可以找到master的日志，这里就是mysql的数据目录
master_ip_failover_script=/etc/mha/script/master_ip_failover  # 设置自动failover时候的切换脚本
master_ip_online_change_script=/etc/mha/script/master_ip_failover  #设置手动failover时候的切换脚本
report_script=/etc/mha/script/sendEmail    # 设置发生切换后发送的报警的脚本
remote_workdir=/tmp    # 设置远端mysql在发生切换时binlog的保存位置
ping_interval=3        #设置监控主库，发送ping包的时间间隔，默认是3秒，尝试三次没有回应的时候自动进行railover

user=mhamon     # 设置监控用户
password=Oracle123    #  设置监控用户的密码

repl_user=repluser        # 设置复制账号
repl_password=Oracle123    # 设置复制账号的密码

ssh_user=root      # 设置ssh的登陆用户
ssh_port=10022           # 设置ssh的端口号


secondary_check_script=/usr/local/bin/masterha_secondary_check -s slave87 -s slave88

[server1]
hostname=192.168.0.175
port=3306
master_binlog_dir=/u01/mysql/mysql_data

[server2]
hostname=192.168.0.176
port=3306
master_binlog_dir=/u01/mysql/mysql_data


[server3]
hostname=192.168.0.200
port=3306
master_binlog_dir=/u01/mysql/mysql_data

```


### 2.5.2 failover脚本的设置
```
#!/usr/bin/env perl

use strict;
use warnings FATAL => 'all';

use Getopt::Long;

my (
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);

my $vip = '192.168.0.177/24';   # 设置VIP
my $key = '88';             # 此处代表 绑定在 eth2:88上
my $ssh_start_vip = "/sbin/ifconfig eth2:$key $vip";  # 开启VIP
my $ssh_stop_vip = "/sbin/ifconfig eth2:$key down";  # 关闭VIP

GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);

exit &main();

sub main {

    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";

    if ( $command eq "stop" || $command eq "stopssh" ) {

        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {

        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}

sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
     return 0  unless  ($ssh_user);
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}

sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_i
p=ip --new_master_port=port\n";
}
```


### 2.5.3 sendmail脚本的设置
没有需求,找了个sample作为参考
```
#!/usr/bin/perl

#  Copyright (C) 2011 DeNA Co.,Ltd.
#
#  This program is free software; you can redistribute it and/or modify
#  it under the terms of the GNU General Public License as published by
#  the Free Software Foundation; either version 2 of the License, or
#  (at your option) any later version.
#
#  This program is distributed in the hope that it will be useful,
#  but WITHOUT ANY WARRANTY; without even the implied warranty of
#  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#  GNU General Public License for more details.
#
#  You should have received a copy of the GNU General Public License
#   along with this program; if not, write to the Free Software
#  Foundation, Inc.,
#  51 Franklin Street, Fifth Floor, Boston, MA  02110-1301  USA

## Note: This is a sample script and is not complete. Modify the script based on your environment.

use strict;
use warnings FATAL => 'all';
use Mail::Sender;
use Getopt::Long;

#new_master_host and new_slave_hosts are set only when recovering master succeeded
my ( $dead_master_host, $new_master_host, $new_slave_hosts, $subject, $body );
my $smtp='smtp.163.com';
my $mail_from='xxx@163.com';
my $mail_user='xxx@163.com';
my $mail_pass='xxxxxx';
my $mail_to=['xxx@xxx.cn','xxxxx@xxx.cn'];
GetOptions(
  'orig_master_host=s' => \$dead_master_host,
  'new_master_host=s'  => \$new_master_host,
  'new_slave_hosts=s'  => \$new_slave_hosts,
  'subject=s'          => \$subject,
  'body=s'             => \$body,
);

mailToContacts($smtp,$mail_from,$mail_user,$mail_pass,$mail_to,$subject,$body);

sub mailToContacts {
    my ( $smtp, $mail_from, $user, $passwd, $mail_to, $subject, $msg ) = @_;
    open my $DEBUG, "> /tmp/monitormail.log"
        or die "Can't open the debug      file:$!\n";
    my $sender = new Mail::Sender {
        ctype       => 'text/plain; charset=utf-8',
        encoding    => 'utf-8',
        smtp        => $smtp,
        from        => $mail_from,
        auth        => 'LOGIN',
        TLS_allowed => '0',
        authid      => $user,
        authpwd     => $passwd,
        to          => $mail_to,
        subject     => $subject,
        debug       => $DEBUG
    };

    $sender->MailMsg(
        {   msg   => $msg,
            debug => $DEBUG
        }
    ) or print $Mail::Sender::Error;
    return 1;
}



# Do whatever you want here

exit 0;
```
### 2.5.4 mha启动脚本
```
# cat bin/mhaCli.sh
#!/bin/bash

. /etc/profile
. ~/.bash_profile
. ~/.bashrc

run_num=$(ps -ef|grep masterha_manager |grep -v grep|wc -l)
pid_file='/data/mha/app1/app1.master_status.health'

start() {
   if [[ $run_num < 1 ]];then
   args="--conf=/data/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover"
   nohup masterha_manager $args < /dev/null > /data/mha/app1/app1.log 2>&1 &
   else
     echo 'mha is already running...'
   fi
}

stop() {

  if [[ $run_num < 1 ]];then
     echo 'mha not running ...'
     exit 64
  else
     ps -ef|grep masterha_manager |grep -v grep|awk '{print $2}'|xargs kill -9
     rm -f $pid_file
     echo 'mha stop...'
  fi
}

status() {
 masterha_check_status --conf=/data/mha/app1.cnf
}

case "$1" in
  start)
  start
  ;;
  stop)
  stop
  ;;
  status)
  status
  ;;
  *)
  echo 'mhaCli {stop|start|status}'
  ;;
esac
```

## 2.6 互通性和复制的验证
### 2.6.1 互通性的验证
验证方法:
```
shell> masterha_check_ssh --conf=/etc/mha/app1/app1.cnf
```
报错:
```
[root@ai2018 .ssh]# masterha_check_ssh --conf=/etc/mha/app1/app1.cnf
Mon Mar 26 22:36:57 2018 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Mon Mar 26 22:36:57 2018 - [info] Reading application default configuration from /etc/mha/app1/app1.cnf..
Mon Mar 26 22:36:57 2018 - [info] Reading server configuration from /etc/mha/app1/app1.cnf..
Mon Mar 26 22:36:57 2018 - [info] Starting SSH connection tests..
Mon Mar 26 22:36:58 2018 - [debug]
Mon Mar 26 22:36:57 2018 - [debug]  Connecting via SSH from root@192.168.0.175(192.168.0.175:10022) to root@192.168.0.176(192.168.0.176:10022)..
Mon Mar 26 22:36:57 2018 - [debug]   ok.
Mon Mar 26 22:36:57 2018 - [debug]  Connecting via SSH from root@192.168.0.175(192.168.0.175:10022) to root@192.168.0.200(192.168.0.200:10022)..
Mon Mar 26 22:36:58 2018 - [debug]   ok.
Mon Mar 26 22:36:58 2018 - [debug]
Mon Mar 26 22:36:57 2018 - [debug]  Connecting via SSH from root@192.168.0.176(192.168.0.176:10022) to root@192.168.0.175(192.168.0.175:10022)..
Mon Mar 26 22:36:58 2018 - [debug]   ok.
Mon Mar 26 22:36:58 2018 - [debug]  Connecting via SSH from root@192.168.0.176(192.168.0.176:10022) to root@192.168.0.200(192.168.0.200:10022)..
Mon Mar 26 22:36:58 2018 - [debug]   ok.
Mon Mar 26 22:36:58 2018 - [error][/usr/local/share/perl5/MHA/SSHCheck.pm, ln63]
Mon Mar 26 22:36:58 2018 - [debug]  Connecting via SSH from root@192.168.0.200(192.168.0.200:10022) to root@192.168.0.175(192.168.0.175:10022)..
Permission denied (publickey,password,keyboard-interactive).
Mon Mar 26 22:36:58 2018 - [error][/usr/local/share/perl5/MHA/SSHCheck.pm, ln111] SSH connection from root@192.168.0.200(192.168.0.200:10022) to root@192.168.0.175(192.168.0.175:10022) failed!
SSH Configuration Check Failed!
```

正确配置后,验证应该如下:
```
[root@ai2018 ~]# masterha_check_ssh --conf=/etc/mha/app1/app1.cnf
Mon Mar 26 23:01:32 2018 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Mon Mar 26 23:01:32 2018 - [info] Reading application default configuration from /etc/mha/app1/app1.cnf..
Mon Mar 26 23:01:32 2018 - [info] Reading server configuration from /etc/mha/app1/app1.cnf..
Mon Mar 26 23:01:32 2018 - [info] Starting SSH connection tests..
Mon Mar 26 23:01:33 2018 - [debug]
Mon Mar 26 23:01:32 2018 - [debug]  Connecting via SSH from root@192.168.0.175(192.168.0.175:10022) to root@192.168.0.176(192.168.0.176:10022)..
Mon Mar 26 23:01:33 2018 - [debug]   ok.
Mon Mar 26 23:01:33 2018 - [debug]  Connecting via SSH from root@192.168.0.175(192.168.0.175:10022) to root@192.168.0.200(192.168.0.200:10022)..
Mon Mar 26 23:01:33 2018 - [debug]   ok.
Mon Mar 26 23:01:34 2018 - [debug]
Mon Mar 26 23:01:33 2018 - [debug]  Connecting via SSH from root@192.168.0.176(192.168.0.176:10022) to root@192.168.0.175(192.168.0.175:10022)..
Mon Mar 26 23:01:33 2018 - [debug]   ok.
Mon Mar 26 23:01:33 2018 - [debug]  Connecting via SSH from root@192.168.0.176(192.168.0.176:10022) to root@192.168.0.200(192.168.0.200:10022)..
Mon Mar 26 23:01:33 2018 - [debug]   ok.
Mon Mar 26 23:01:34 2018 - [debug]
Mon Mar 26 23:01:33 2018 - [debug]  Connecting via SSH from root@192.168.0.200(192.168.0.200:10022) to root@192.168.0.175(192.168.0.175:10022)..
Mon Mar 26 23:01:34 2018 - [debug]   ok.
Mon Mar 26 23:01:34 2018 - [debug]  Connecting via SSH from root@192.168.0.200(192.168.0.200:10022) to root@192.168.0.176(192.168.0.176:10022)..
Mon Mar 26 23:01:34 2018 - [debug]   ok.
Mon Mar 26 23:01:34 2018 - [info] All SSH connection tests passed successfully.
```
### 2.6.2 复制正确性的验证

报错1
```
[root@ai2018 app1]# masterha_check_repl --conf=/etc/mha/app1/app1.cnf
Mon Mar 26 22:51:18 2018 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Mon Mar 26 22:51:18 2018 - [info] Reading application default configuration from /etc/mha/app1/app1.cnf..
Mon Mar 26 22:51:18 2018 - [info] Reading server configuration from /etc/mha/app1/app1.cnf..
Mon Mar 26 22:51:18 2018 - [info] MHA::MasterMonitor version 0.57.
Mon Mar 26 22:51:18 2018 - [info] GTID failover mode = 1
Mon Mar 26 22:51:18 2018 - [info] Dead Servers:
Mon Mar 26 22:51:18 2018 - [info] Alive Servers:
Mon Mar 26 22:51:18 2018 - [info]   192.168.0.175(192.168.0.175:3306)
Mon Mar 26 22:51:18 2018 - [info]   192.168.0.176(192.168.0.176:3306)
Mon Mar 26 22:51:18 2018 - [info]   192.168.0.200(192.168.0.200:3306)
Mon Mar 26 22:51:18 2018 - [info] Alive Slaves:
Mon Mar 26 22:51:18 2018 - [info]   192.168.0.176(192.168.0.176:3306)  Version=5.7.21-log (oldest major version between slaves) log-bin:enabled
Mon Mar 26 22:51:18 2018 - [info]     GTID ON
Mon Mar 26 22:51:18 2018 - [info]     Replicating from 192.168.0.175(192.168.0.175:3306)
Mon Mar 26 22:51:18 2018 - [info]   192.168.0.200(192.168.0.200:3306)  Version=5.7.21-log (oldest major version between slaves) log-bin:enabled
Mon Mar 26 22:51:18 2018 - [info]     GTID ON
Mon Mar 26 22:51:18 2018 - [info]     Replicating from 192.168.0.175(192.168.0.175:3306)
Mon Mar 26 22:51:18 2018 - [info] Current Alive Master: 192.168.0.175(192.168.0.175:3306)
Mon Mar 26 22:51:18 2018 - [info] Checking slave configurations..
Mon Mar 26 22:51:18 2018 - [info] Checking replication filtering settings..
Mon Mar 26 22:51:18 2018 - [info]  binlog_do_db= , binlog_ignore_db=
Mon Mar 26 22:51:18 2018 - [info]  Replication filtering check ok.
Mon Mar 26 22:51:18 2018 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln427] Error happened on checking configurations. Got MySQL error when checking replication privilege. 29: File './mysql/user.MYD' not found (Errcode: 2 - No such file or directory) query:SELECT Repl_slave_priv AS Value FROM mysql.user WHERE user = ?
 at /usr/local/share/perl5/MHA/Server.pm line 397
Mon Mar 26 22:51:18 2018 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln525] Error happened on monitoring servers.
Mon Mar 26 22:51:18 2018 - [info] Got exit code 1 (Not master dead).

MySQL Replication Health is NOT OK!
```

检查过程
```
mysql> SELECT Repl_slave_priv AS Value FROM mysql.user WHERE user = 'repluser';
ERROR 29 (HY000): File './mysql/user.MYD' not found (Errcode: 2 - No such file or directory)
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
+--------------------+
1 row in set (0.00 sec)

mysql> exit
```

报错2
```
[root@ai2018 ~]# masterha_check_repl  --conf=/etc/mha/app1/app1.cnf
Mon Mar 26 23:01:44 2018 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Mon Mar 26 23:01:44 2018 - [info] Reading application default configuration from /etc/mha/app1/app1.cnf..
Mon Mar 26 23:01:44 2018 - [info] Reading server configuration from /etc/mha/app1/app1.cnf..
Mon Mar 26 23:01:44 2018 - [info] MHA::MasterMonitor version 0.57.
Mon Mar 26 23:01:44 2018 - [info] GTID failover mode = 1
Mon Mar 26 23:01:44 2018 - [info] Dead Servers:
Mon Mar 26 23:01:44 2018 - [info] Alive Servers:
Mon Mar 26 23:01:44 2018 - [info]   192.168.0.175(192.168.0.175:3306)
Mon Mar 26 23:01:44 2018 - [info]   192.168.0.176(192.168.0.176:3306)
Mon Mar 26 23:01:44 2018 - [info]   192.168.0.200(192.168.0.200:3306)
Mon Mar 26 23:01:44 2018 - [info] Alive Slaves:
Mon Mar 26 23:01:44 2018 - [info]   192.168.0.176(192.168.0.176:3306)  Version=5.7.21-log (oldest major version between slaves) log-bin:enabled
Mon Mar 26 23:01:44 2018 - [info]     GTID ON
Mon Mar 26 23:01:44 2018 - [info]     Replicating from 192.168.0.175(192.168.0.175:3306)
Mon Mar 26 23:01:44 2018 - [info]   192.168.0.200(192.168.0.200:3306)  Version=5.7.21-log (oldest major version between slaves) log-bin:enabled
Mon Mar 26 23:01:44 2018 - [info]     GTID ON
Mon Mar 26 23:01:44 2018 - [info]     Replicating from 192.168.0.175(192.168.0.175:3306)
Mon Mar 26 23:01:44 2018 - [info] Current Alive Master: 192.168.0.175(192.168.0.175:3306)
Mon Mar 26 23:01:44 2018 - [info] Checking slave configurations..
Mon Mar 26 23:01:44 2018 - [info]  read_only=1 is not set on slave 192.168.0.200(192.168.0.200:3306).
Mon Mar 26 23:01:44 2018 - [info] Checking replication filtering settings..
Mon Mar 26 23:01:44 2018 - [info]  binlog_do_db= , binlog_ignore_db=
Mon Mar 26 23:01:44 2018 - [info]  Replication filtering check ok.
Mon Mar 26 23:01:44 2018 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Mon Mar 26 23:01:44 2018 - [info] Checking SSH publickey authentication settings on the current master..
Mon Mar 26 23:01:44 2018 - [info] HealthCheck: SSH to 192.168.0.175 is reachable.
Mon Mar 26 23:01:44 2018 - [info]
192.168.0.175(192.168.0.175:3306) (current master)
 +--192.168.0.176(192.168.0.176:3306)
 +--192.168.0.200(192.168.0.200:3306)

Mon Mar 26 23:01:44 2018 - [info] Checking replication health on 192.168.0.176..
Mon Mar 26 23:01:44 2018 - [info]  ok.
Mon Mar 26 23:01:44 2018 - [info] Checking replication health on 192.168.0.200..
Mon Mar 26 23:01:44 2018 - [info]  ok.
Mon Mar 26 23:01:44 2018 - [info] Checking master_ip_failover_script status:
Mon Mar 26 23:01:44 2018 - [info]   /etc/mha/script/master_ip_failover --command=status --ssh_user=root --orig_master_host=192.168.0.175 --orig_master_ip=192.168.0.175 --orig_master_port=3306  --orig_master_ssh_port=10022
Mon Mar 26 23:01:44 2018 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln427] Error happened on checking configurations. Can't exec "/etc/mha/script/master_ip_failover": Permission denied at /usr/local/share/perl5/MHA/ManagerUtil.pm line 68.
Mon Mar 26 23:01:44 2018 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln525] Error happened on monitoring servers.
Mon Mar 26 23:01:44 2018 - [info] Got exit code 1 (Not master dead).

MySQL Replication Health is NOT OK!
Mon Mar 26 23:01:44 2018 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln229]  Failed to get master_ip_failover_script status with return code 1:0.
Mon Mar 26 23:01:44 2018 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln427] Error happened on checking configurations.  at /usr/local/bin/masterha_check_repl line 48
Mon Mar 26 23:01:44 2018 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln525] Error happened on monitoring servers.
Mon Mar 26 23:01:44 2018 - [info] Got exit code 1 (Not master dead).

MySQL Replication Health is NOT OK!
```

正确的设置后,验证通过应该如下:
```
[root@ai2018 ~]# masterha_check_repl  --conf=/etc/mha/app1/app1.cnf
Mon Mar 26 23:02:21 2018 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Mon Mar 26 23:02:21 2018 - [info] Reading application default configuration from /etc/mha/app1/app1.cnf..
Mon Mar 26 23:02:21 2018 - [info] Reading server configuration from /etc/mha/app1/app1.cnf..
Mon Mar 26 23:02:21 2018 - [info] MHA::MasterMonitor version 0.57.
Mon Mar 26 23:02:21 2018 - [info] GTID failover mode = 1
Mon Mar 26 23:02:21 2018 - [info] Dead Servers:
Mon Mar 26 23:02:21 2018 - [info] Alive Servers:
Mon Mar 26 23:02:21 2018 - [info]   192.168.0.175(192.168.0.175:3306)
Mon Mar 26 23:02:21 2018 - [info]   192.168.0.176(192.168.0.176:3306)
Mon Mar 26 23:02:21 2018 - [info]   192.168.0.200(192.168.0.200:3306)
Mon Mar 26 23:02:21 2018 - [info] Alive Slaves:
Mon Mar 26 23:02:21 2018 - [info]   192.168.0.176(192.168.0.176:3306)  Version=5.7.21-log (oldest major version between slaves) log-bin:enabled
Mon Mar 26 23:02:21 2018 - [info]     GTID ON
Mon Mar 26 23:02:21 2018 - [info]     Replicating from 192.168.0.175(192.168.0.175:3306)
Mon Mar 26 23:02:21 2018 - [info]   192.168.0.200(192.168.0.200:3306)  Version=5.7.21-log (oldest major version between slaves) log-bin:enabled
Mon Mar 26 23:02:21 2018 - [info]     GTID ON
Mon Mar 26 23:02:21 2018 - [info]     Replicating from 192.168.0.175(192.168.0.175:3306)
Mon Mar 26 23:02:21 2018 - [info] Current Alive Master: 192.168.0.175(192.168.0.175:3306)
Mon Mar 26 23:02:21 2018 - [info] Checking slave configurations..
Mon Mar 26 23:02:21 2018 - [info]  read_only=1 is not set on slave 192.168.0.200(192.168.0.200:3306).
Mon Mar 26 23:02:21 2018 - [info] Checking replication filtering settings..
Mon Mar 26 23:02:21 2018 - [info]  binlog_do_db= , binlog_ignore_db=
Mon Mar 26 23:02:21 2018 - [info]  Replication filtering check ok.
Mon Mar 26 23:02:21 2018 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Mon Mar 26 23:02:21 2018 - [info] Checking SSH publickey authentication settings on the current master..
Mon Mar 26 23:02:21 2018 - [info] HealthCheck: SSH to 192.168.0.175 is reachable.
Mon Mar 26 23:02:21 2018 - [info]
192.168.0.175(192.168.0.175:3306) (current master)
 +--192.168.0.176(192.168.0.176:3306)
 +--192.168.0.200(192.168.0.200:3306)

Mon Mar 26 23:02:21 2018 - [info] Checking replication health on 192.168.0.176..
Mon Mar 26 23:02:21 2018 - [info]  ok.
Mon Mar 26 23:02:21 2018 - [info] Checking replication health on 192.168.0.200..
Mon Mar 26 23:02:21 2018 - [info]  ok.
Mon Mar 26 23:02:21 2018 - [info] Checking master_ip_failover_script status:
Mon Mar 26 23:02:21 2018 - [info]   /etc/mha/script/master_ip_failover --command=status --ssh_user=root --orig_master_host=192.168.0.175 --orig_master_ip=192.168.0.175 --orig_master_port=3306  --orig_master_ssh_port=10022
Unknown option: orig_master_ssh_port


IN SCRIPT TEST====/sbin/ifconfig eth2:91 down==/sbin/ifconfig eth2:91 192.168.0.177/24===

Checking the Status of the script.. OK
Mon Mar 26 23:02:21 2018 - [info]  OK.
Mon Mar 26 23:02:21 2018 - [warning] shutdown_script is not defined.
Mon Mar 26 23:02:21 2018 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.
```

## 2.7 启动manager节点：
```
shell>  /etc/mha/bin/mhaCli.sh start
```
　　
## 2.8 检查mha manager的状态：
```
shell> masterha_check_status --conf=/etc/mha/app1/app1.cnf
或者： /etc/mha/bin/mhaCli.sh status
```

## 2.9 配置VIP,添加虚拟Ip地址
```
[root@nazeebo mysql_data]# ifconfig eth2:88 192.168.0.177/24
[root@nazeebo mysql_data]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 00:50:56:a5:b0:38 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.175/24 brd 192.168.0.255 scope global eth2
    inet 192.168.0.177/24 brd 192.168.0.255 scope global secondary eth2:88
    inet6 fe80::250:56ff:fea5:b038/64 scope link
       valid_lft forever preferred_lft forever
[root@nazeebo mysql_data]#
```

# 3. 验证切换验证
将master节点的数据库停掉,过一会儿看vip是否飘移到新的master的ethx上

# 4.节点重新上下步骤
当出问题的原master服务器175问题修复好后，此时需要重新上线主机，则可以通过以下方式：

1. 在服务器175上搭建好mysql服务，建议和之前配置参数一致；服务器之间免密。
2. 在现在的master或者slave使用mysqldump将数据备份，加--master-data=2 -A参数
3. 将备份数来的数据在服务器175上进行恢复，完成后执行flush privileges刷新权限。
4. 成后配置GTID的change master操作，start slave即可
5. 将主机的信息添加到mha的配置文件中，以便mha manager检测到新的节点主机
6. 使用mha的测试命令进行测试，成功则启动mha程序即可



# 5.其他
