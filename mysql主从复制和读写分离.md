# mysql主从集群配置
主从集群配置和读写分离技术是mysql服务器常见优化方案：主服务器处理写入、修改、删除操作，从服务器处理读取操作，并且同步主服务器的操作，从而缓解主服务器压力。
   
### 1. 原理
* 主服务器Master：建立二进制日志，每当主服务器执行sql语句(或者产生磁盘变化)时，写进二进制日志；
* 从服务器Slave：建立relay-log，并利用主服务器授权的复制账号，来监听主服务器的日志变化，从而进行同步
   
### 2. 配置步骤
   
(1). 修改Master的/etc/my.cnf文件
```
#server-id
server-id=5001

#binary log
log-bin=master-mysql-bin

#二进制日志格式
#statement | row | mixed
binlog-format=mixed  #当sql语句只影响单行数据时，采用row格式；否则适合用statement格式日志
```
(2). 修改Slave的/etc/my.cnf文件
```
#server-id
server-id=5002

#relay log
relay-log=slave-mysql-relay
```
(3). 主服务器建立复制授权账号：master_user='repl'、master_password='123456'
```
mysql> grant replication client, replication slave on *.* to 'repl'@'192.168.%.%' identified by '123456';
```
(4). 通过show master status查询主服务器日志文件和位置，通过指定要复制的主服务器和所授权的账号、密码，监听主服务器二进制日志：
```
mysql> change master to master_host='192.168.0.199',  master_user='repl', master_password='123456', master_port=3306, master_log_file='master-mysql-bin.000001', master_log_pos=278, master_connect_retry=30;
```
(5). 常用语句
```
show master status; #查看master状态：查看当前服务器的日志、位置
show slave status; #查看slave状态
reset slave; #重置slave状态
start slave; #启动slave状态(开始监听master变化)
stop slave; #暂停slave状态
```
### 3. 主主复制
有时为了减轻主服务器压力，会配置多台主服务器，配置思路类似主从复制配置，以两台为例：
* 都设置二进制日志和relay日志
* 都设置复制账号
* 都设置对方为自己的master
### 4. 主主复制的陷阱和技巧
当主键使用自增id时，容易出现id冲突问题，解决方案
(1). 服务器数目可以确定时，比如确定只有2台主服，则可以设置主键自动增长步长，一个奇数增长、一个偶数增长
```
修改配置文件
MasterA：set global auto_increment_increment=2; set global auto_increment_offset=1
MasterB：set global auto_increment_increment=2; set global auto_increment_offset=2
```
(2). 服务器数目不确定时，可以根据服务器编号设置比较大的基数
```
5001号：set global auto_increment_increment=50000000;
6001号：set global auto_increment_increment=60000000;
```
### 5. 被动模式的主主复制
为了方便快速处理服务器故障，当某台Master出故障时，既不想停服，又不想将slave提升为Master；可以部署2台Master，
其中一台Master设为只读，当可写Master出故障时，可以快速将只读Master切换为可写模式，从而做到无缝切换。
```
修改配置文件
read only=1; 只读模式
```

# mysql负载均衡和读写分离
完成数据库的主从复制配置，可以继续利用读写分离技术来继续提升数据库的并发、负载能力；如下图所示：

![读写分离](http://heylinux.com/wp-content/uploads/2011/06/mysql-master-salve-proxy.jpg)

mysql读写分离和负载均衡通常采用中间件工具实现，如mysql-proxy、mycat、mysql router、Amoeba for mysql，这里只简要介绍2款
### 1. mysql-proxy
mysql-proxy是mysql官方提供的一款中间件，是一个处于client端和mysql server端之间简单程序，负责将client端的请求转发给后台数据库，通过使用rw-splitting.lua脚本进行分析，实现复杂的连接控制和过滤，从而实现读写分离和负载平衡。
   
(1). mysql-proxy.cnf配置：
```
[mysql-proxy] 
admin-username=user_name     #admin用户名
admin-password=password      #admin密码
proxy-address=0.0.0.0:3307   #代理的监听地址端口，默认端口4040
proxy-backend-addresses=192.168.1.101:3306 #指定主master用于写入数据
proxy-read-only-backend-addresses=192.168.1.102:3306 #指定slave用于读取数据

admin-lua-script=/usr/lib64/mysql-proxy/lua/admin.lua       #lua管理脚本路径
proxy-lua-script=/opt/mysql-proxy/scripts/rw-splitting.lua  #lua读写分离脚本路径
daemon=true     #mysql-proxy以守护进程模式启动
keepalive=true  #使进程在异常关闭后能够自动恢复
log-level=warning #定义log日志级别，由高到低分别有(error|warning|info|message|debug)
log-file=/opt/mysql-proxy/log/mysql-proxy.log  #log日志文件路径
```
(2). 修改读写分离脚本rw-splitting.lua默认配置
```lua
if not proxy.global.config.rwsplit then
proxy.global.config.rwsplit = {
min_idle_connections = 1, //默认为4, 默认要达到连接数为4时才启用读写分离
max_idle_connections = 1, //默认为8
is_debug = false
}
```

(3). mysql-proxy缺点：

mysql-proxy不稳定，在高并发或有错误连接的情况下，进程很容易自动关闭，通常设置keepalive为True，让进程自动恢复是个比较好的办法；
最稳妥的做法是在每个从服务器上安装一个mysql-proxy供自身使用，虽然比较低效但却能保证稳定性；

### 2. Amoeba for mysql
Amoeba for Mysql是一款优秀的中间件软件，同样可以实现读写分离, 负载均衡、充当sql路由等功能, 稳定性远远超过mysql-proxy。架构原理图如下：

![Amoeba for Mysql架构图](https://www.centos.bz/wp-content/uploads/2012/05/r-w-splitting.jpg)

















