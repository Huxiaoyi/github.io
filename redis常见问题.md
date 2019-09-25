时间：2019-09

# Redis常见问题
## Redis启动问题
1. ResponseError：Client sent AUTH，but no password is set
<br/> redis安装配置文件(6379.conf)连接密码设置，与客户端建立redis连接密码设置不一致是redis常见问题。
<br/> 有时候配置文件确实没有设置密码，这种情况，去掉客户端连接密码即可；如果配置文件设置了密码，无论怎么重启还是报同样错误，试试指定配置文件重启：
<br/>①. 如果是用apt-get或者yum install安装的redis，可以直接通过下面的命令关闭redis
<br/>/etc/init.d/redis-server stop
<br/>
<br/>②. 如果是通过源码安装的redis，则可以通过redis的客户端程序redis-cli的shutdown命令来关闭redis
<br/>redis-cli -h 127.0.0.1 -p 6379 shutdown
<br/>
<br/>③. 如果上述方式都没有成功停止redis，则可以使用终极武器 kill -9
<br/>
<br/>④. redis启动
<br/>redis-server  /etc/redis/7379.conf

