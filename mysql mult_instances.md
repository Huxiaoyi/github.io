时间：2019-07

作者：

# MySQL Multiple Instances

mysql多实例，简单说就是在一台服务器上，配置多份mysql配置文件，配置完毕后，可以在一台服务器上同时运行这些mysql服务进程，每个服务进程通过不同的socket监听不同的服务端口，来提供各自的服务。 此外可以对这些mysql实例进行主从配置，当业务需求不大时，可以节省服务器资源达到充分利用资源目的。

## 配置环境
系统：Linux hujiayi-virtual-machine 4.18.0-25-generic #26~18.04.1-Ubuntu SMP x86_64 GNU/Linux
<br/>mysql：Distrib 5.7.27，端口号3306，实例1端口号为3304，实例2端口号为3305，最终结果如下：
```bash
root@hujiayi-virtual-machine:/usr/bin# netstat -ntulp | grep mysqld
tcp6       0      0 :::3303                 :::*                    LISTEN      5828/mysqld         
tcp6       0      0 :::3304                 :::*                    LISTEN      6544/mysqld         
tcp6       0      0 :::3306                 :::*                    LISTEN      1501/mysqld 
```

## 创建mysql多实例步骤
①. 在var/lib目录下，创建保存多实例目录，并赋予目录mysql权限
```bash
root@hujiayi-virtual-machine:/var/lib# mkdir mysql_3303
root@hujiayi-virtual-machine:/var/lib# mkdir mysql_3304
```
赋予mysql_3303和mysql_3304目录mysql权限
```bash
root@hujiayi-virtual-machine:/var/lib# chown -R mysql.mysql mysql_3303
root@hujiayi-virtual-machine:/var/lib# chown -R mysql.mysql mysql_3304
```
②. 在apparmor中添加读写权限，并重启apparmor服务
```bash
root@hujiayi-virtual-machine:/etc/apparmor.d# vim usr.sbin.mysqld
```
添加下面内容
```vim
/var/lib/mysql_3303/ r,
/var/lib/mysql_3303/** rwk,

/var/lib/mysql_3304/ r,
/var/lib/mysql_3304/** rwk,
```
重启apparmor服务
```bash
root@hujiayi-virtual-machine:/etc/apparmor.d# service apparmor restart
```
③. 配置多实例数据库配置文件
<br/>将my.cnf文件拷贝两份，分别为my_3003.cnf和my_3004.cnf；
```bash
root@hujiayi-virtual-machine:/etc/mysql# cp my.cnf my_3303.cnf
root@hujiayi-virtual-machine:/etc/mysql# cp my.cnf my_3304.cnf
```
修改配置文件port、socket、pid-file、datadir、log_error字段，以my_3303.cnf为例
```vim
[client]
port        = 3303
socket      = /var/lib/mysql_3303/mysqld.sock

[mysqld_safe]
socket      = /var/lib/mysql_3303/mysqld.sock

[mysqld]
user        = mysql
pid-file    = /var/lib/mysql_3303/mysqld.pid
socket      = /var/lib/mysql_3303/mysqld.sock
port        = 3303
basedir     = /usr
datadir     = /var/lib/mysql_3303
tmpdir      = /tmp

#nice       =0
server-id	= 3303
relay-log-index = slave-relay-bin.index
relay-log	= slave-relay-bin
log_error   = /var/lib/mysql_3303/error.log
```
④. 初始化mysql多实例，以my_3303.cnf为例
```bash
root@hujiayi-virtual-machine:/etc/mysql# cd /usr/bin/
root@hujiayi-virtual-machine:/usr/bin# mysql_install_db  --defaults-file=/etc/mysql/my_3303.cnf   --basedir=/usr/  --datadir=/var/lib/mysql_3303 --user=mysql
```
⑤. 重新设置mysql实例密码，设置直接跳过密码进入mysql，以my_3303.cnf为例
```bash
root@hujiayi-virtual-machine:/usr/bin# mysqld_safe --datadir=/var/lib/mysql_3303 --pid-file=/var/lib/mysql_3303/mysqld.pid --socket=/var/lib/mysql_3303/mysqld.sock --port=3303 --skip-grant-tables &id --socket=/var/lib/mysql_3303/mysqld.sock --port=3303 --skip-grant-table
[1] 4310
root@hujiayi-virtual-machine:/usr/bin# 2019-08-04T06:55:55.921342Z mysqld_safe Logging to syslog.
2019-08-04T06:55:55.939088Z mysqld_safe Logging to '/var/log/mysql/error.log'.
2019-08-04T06:55:55.978658Z mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql_3303
```

设置完毕后，可以无密码进入mysql，提示输入密码时，直接回车即可，以my_3303.cnf为例
```bash
root@hujiayi-virtual-machine:/usr/bin# mysql -uroot -p -S /var/lib/mysql_3303/mysqld.sock
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.27-0ubuntu0.18.04.1-log (Ubuntu)
```

⑥. 查询密码
```bash
mysql> select user,host,authentication_string from mysql.user\G;
*************************** 1. row ***************************
                 user: mysql.session
                 host: localhost
authentication_string: *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE
*************************** 2. row ***************************
                 user: root
                 host: localhost
authentication_string: *0BB0E18B1FB4192422E09C8368163FF30497038F
*************************** 3. row ***************************
                 user: mysql.sys
                 host: localhost
authentication_string: *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE
3 rows in set (0.03 sec)
```
⑦. 重置密码：
```bash
mysql> update mysql.user set authentication_string=password("123456") where user='root';

mysql> set password = password('123456');
Query OK, 0 rows affected, 1 warning (0.02 sec)
刷新退出
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> exit;
Bye
```
⑧. 启动mysql多实例，以my_3303.cnf为例
```bash
root@hujiayi-virtual-machine:/usr/bin# mysqld_safe --defaults-file=/etc/mysql/my_3303.cnf
```
如果一直卡在那里不动，说明mysql多实例启动成功，测试端口是否监听，如果有则说明启动成功
```bash
root@hujiayi-virtual-machine:/usr/bin# netstat -nlt | grep 3303
tcp6       0      0 :::3303                 :::*                    LISTEN    
```
或者查看mysql服务进程
```bash
root@hujiayi-virtual-machine:/usr/bin# netstat -ntulp | grep mysqld
tcp6       0      0 :::3303                 :::*                    LISTEN      5828/mysqld         
tcp6       0      0 :::3306                 :::*                    LISTEN      1501/mysqld  
```

⑨. 关闭mysql多实例，以3303为例
```bash
root@hujiayi-virtual-machine:/usr/bin# mysqladmin -u root -p123456  -S /var/lib/mysql_3303/mysqld.sock shutdown
mysqladmin: [Warning] Using a password on the command line interface can be insecure.
```

⑩. 设置可远程连接
```bash
mysql> grant all privileges on  *.* to 'root'@'%' identified by '123456' with grant option;
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```
设置完后，就可以使用SQLyog等第三方可视化工具连接数据库了

