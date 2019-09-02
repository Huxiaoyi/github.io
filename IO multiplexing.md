时间：2019-06

作者：

# IO multiplexing
假设服务器需要处理1000个连接，一般需要1000个线程或进程来处理这1000个连接，而1000个线程大部分是被阻塞起来的。由于CPU的核数有限制，系统能为每个线程分配的时间片就会非常短，而线程尤其进程的切换频繁时，是需要很大开销的。 于是引入了IO multiplexing技术，即IO多路复用，指内核通过监视多个描述符的读、写等事件，一旦某个描述符就绪，就能够将发生的事件通知给关心的应用程序去处理该事件。 同样处理1000个连接时，只需要1个线程监控就绪状态，对就绪的每个连接开一个线程处理就可以了，这样需要的线程数大大减少，减少了内存开销和上下文切换的CPU开销。常见的IO多路复用的机制有select、poll、epoll。

## 同步、异步、阻塞与非阻塞
先了解几个常用却容易混淆的基本概念：
①. 同步与异步
同步与异步关注的是消息通知的机制：同步情况下，是由消息处理者自己亲自等待消息是否被触发，而异步的情况下，是由触发机制来通知消息处理者。
举个烧开水的例子，同步方式情况下，烧水者一直盯着水是否煮沸；而异步方式相当于用一个水烧开了会鸣叫的水壶，烧水者开始烧水后就无需关心水的状态，由水壶的自动鸣叫通知烧水者。

②. 阻塞与非阻塞
阻塞与非阻塞关注的是线程在等待消息通知时的状态，即在等待消息通知过程中能不能做别的事情。还是以烧开水为例，阻塞方式就是烧水过程中不做任何事情，专心致志烧水；而非阻塞方式即烧水过程中可以做其它任何事情。  所以，由上面两组情况便会诞生出下面四种组合：
a. 同步阻塞：烧水者一直盯着水，等待水沸腾；效率最低；
b. 同步非阻塞：烧水者一边玩手机，每等待一段时间(比如30秒)后看一眼水是否沸腾；实际上由于频繁地切换状态效率是比较低的；
c. 异步阻塞：烧水者坐等水壶鸣叫，等待水壶鸣叫过程中什么事情也不做；
d. 异步非阻塞：烧水者在等待水壶鸣叫过程中一直玩手机；效率比较高；


## 用户空间与内核空间
操作系统空间的划分是为了保证用户不能直接操作系统内核。以32位操作系统举例，32系统寻址空间为4G，系统为了保证内核的安全，将最高的1G空间划分给内核使用，而将较低剩余的3G空间提供给各个进程使用，即称为用户空间。

## 文件描述符
我们知道Linux很重要的设计思想就是一切皆文件。内核会给每个访问的文件分配一个文件描述符(在打开或新建文件时返回)，读写文件都需要通过这个文件描述符。文件描述符在形式上是一个非负整数。实际上是一个索引值，用于指向(内核为每一个进程维护文件记录表)每个进程打开文件的记录表。

## IO流程
网络IO的本质是socket的读取，以read举例，对于一次IO访问，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。 所以当一个read操作发生时，会经历两个阶段：
第一阶段：等待数据准备阶段：通常涉及等待网络分组数据到达，然后被复制到内核的某个缓冲区；
第二阶段：数据从内核拷贝到进程中：即把数据从内核缓冲区拷贝到应用进程缓冲区。

## IO多路复用
通常情况下，高并发一般离不开多进程和多线程。与多进程和多线程技术相比，IO多路复用技术最大优势是系统开销小，系统不必创建进程和线程，也不必维护这些进程和线程，从而大大减小了系统的开销。 目前支持IO多路复用的系统调用有select，poll和epoll。 IO多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪(一般是读就绪或者写就绪)，能够通知程序进行相应的读写操作。 但select，poll和epoll本质上都是同步IO，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步IO则无需自己负责进行读写，异步IO的实现会负责把数据从内核拷贝到用户空间。 下面分别讲述这三种IO多路复用机制：
①. select
select监视的文件描述符分为三类，即writefds、readfds、和exceptfds。当用户process调用select时候(调用后select函数会阻塞)，select将需要监控的描述符集合拷贝到内核空间，然后遍历自己监控的socket，遍历完所有的socket后，如果没有任何一个socket可读，那么select会调用schedule_timeout进入schedule循环，使得process进入睡眠；如果在timeout时间内某个socket上设备就绪了(有数据 可读、可写、或者有except)，或者超时(timeout可用于指定等待时间，如果需要立即返回，设为null即可)时，则调用select的process会被唤醒，遍历监控的socket集合，找到就绪的描述符并返回给用户； select最大的缺陷是单个进程所能打开的FD是有限制的，由FD_SETSIZE设置，默认值是1024。虽然该值可以修改，重新进行编译，但是并不推荐。此外，对socket进行扫描时是线性扫描，即采用轮询的方法，每次全扫描，效率较低。

②. poll
poll和select非常相似，本质上没有区别，poll并没着手解决性能问题。 poll改变了fds集合的描述方式，使用了pollfd结构而不是select的fd_set结构，这使得poll支持的fds集合限制远大于select的1024。poll虽然解决了fds集合大小上限的限制问题，依然需要将大量描述符数组，整体复制于用户态和内核态之间；以及当个别描述符就绪时，会触发描述符集合全遍历的低效问题。因此poll随着监控的socket集合的增加而性能会线性下降，所以poll不适合用于大并发场景。

③. epoll
epoll是基于select和poll的缺陷提出的增强版本，首先利用mmap映射减少fds集合复制的开销；其二，不再采用轮询的方式，也不再使用全扫描方式，而是通过epoll_ctl注册fd，一旦该fd就绪，内核就会采用类似callback的回调机制来激活该fd，因此不会随着FD数目的增加效率下降。 但是epoll也有缺点，当连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，因为epoll的通知机制需要很多函数回调。 
所以在并发高，连接活跃度不是很高情况下，epoll比select好，例如通常情况下，网站并发较高，但连接活跃度较低，采用epoll较好；当并发性不高，同时连接很活跃时，select比epoll好，比如连接活跃度很高的游戏。

## 通过select+回调+事件循环，用单线程实现高并发
虽然select有效地解决了高并发场景中多线程切换开销大问题，但select是同步+阻塞的IO方式，效率依然不够高，但我们可以通过Loop循环和注册回调机制有效解决这一问题，实现单一线程的select的高并发。下面举一个通过同步方式的例子和通过select+回调+事件循环的例子进行性能比较：
①. select同步阻塞方式
```python
# coding=utf-8
import time
import socket
from urllib.parse import urlparse
from selectors import DefaultSelector, EVENT_READ, EVENT_WRITE

DefaultSelector是以上IO多路复用的某一个子类，它会根据当前系统环境选择最有效的Selector。
selector = DefaultSelector()

def get_url(url):
	# 通过socket请求html
	url = urlparse(url)
	host = url.netloc
	path = url.path
	if path == "":
		path = "/"

	# 建立socket连接
	client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

	# 设置为非阻塞方式，即立即返回
	client.setblocking(False)
	try:
		client.connect((host, 80))  # 建立连接，该过程会阻塞，但不会消耗cpu
	except BlockingIOError as e:
		pass

	# 不停的询问连接是否建立好，建立连接则请求数据
	while True:
		try:
			client.send("GET {} HTTP/1.1\r\nHost:{}\r\nConnection:close\r\n\r\n".format(path, host).encode("utf8"))
			break
		except OSError as e:
			pass

    # 接收数据
	data = b""
	while True:
		try:
			d = client.recv(1024)
		except BlockingIOError as e:
			continue
			
		if d:
			data += d
		else:
			break

    # 打印数据
	data = data.decode("utf8")
	html_data = data.split("\r\n\r\n")[1]
	print(html_data)
	client.close()


if __name__ == "__main__":
	start_time = time.time()
	for url in range(20):
		url = "http://shop.projectsedu.com/goods/{}/".format(url)
		get_url(url)

	print(time.time() - start_time)
```
运行结果：18.801735639572144

②. select+回调+事件循环方式
```python
# coding=utf-8

import time
import socket
from urllib.parse import urlparse
from selectors import DefaultSelector, EVENT_READ, EVENT_WRITE

selector = DefaultSelector()
urls_list = []
stop = False


class Fetcher(object):
	def connected(self, key):
		# 注销连接回调
		selector.unregister(key.fd)
		self.client.send(
			"GET {} HTTP/1.1\r\nHost:{}\r\nConnection:close\r\n\r\n".format(self.path, self.host).encode("utf8"))

		# 注册可读回调
		selector.register(self.client.fileno(), EVENT_READ, self.readable)

	def readable(self, key):
		# 可读, 直接读即可
		global urls_list
		d = self.client.recv(1024)
		if d:
			self.data += d
		else:
			# 注销可读回调
			selector.unregister(key.fd)

			data = self.data.decode("utf8")
			html_data = data.split("\r\n\r\n")[1]
			print(html_data)
			self.client.close()

			urls_list.remove(self.spider_url)
			if not urls_list:
				global stop
				stop = True

	def get_url(self, url):
		self.spider_url = url
		url = urlparse(url)
		self.host = url.netloc
		self.path = url.path
		self.data = b""
		if self.path == "":
			self.path = "/"

		# 建立socket连接
		self.client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

		# 设置为非阻塞方式，立即返回
		self.client.setblocking(False)

		try:
			self.client.connect((self.host, 80))  # 建立连接，该过程会阻塞，但不会消耗cpu
		except BlockingIOError as e:
			pass

		# 注册连接回调
		selector.register(self.client.fileno(), EVENT_WRITE, self.connected)


def Loop():
	global stop
	while not stop:
		# 事件循环，不停的请求socket的状态,并调用对应的回调函数
		ready = selector.select()
		for key, mask in ready:
			call_back = key.data
			call_back(key)


if __name__ == "__main__":
	fetcher = Fetcher()
	start_time = time.time()
	for i in range(20):
		url = "http://shop.projectsedu.com/goods/{}/".format(i)
		urls_list.append(url)
		fetcher = Fetcher()
		fetcher.get_url(url)

	Loop()
	print(time.time() - start_time)
```
运行结果：1.9632670879364014

通过上面例子比较，可以明显看出select+回调+事件循环方式的优越性，实际上协程，tornado、tiwwer、异步IO框架都是基于该方式。
