# Python 多线程、多进程编程和源码简单分析

## Python中的GIL锁
GIL锁，全称Global Interpreter Lock，即全局解释器锁； 正是由它控制着字节码解释器的执行权限，决定了什么时候该进行线程切换；GIL是Python解释器设计的历史遗留问题：为了利用多核，Python开始支持多线程,而解决多线程之间数据完整性和状态同步的最简单方法自然就是加锁。 于是有了GIL这把超级大锁，而当越来越多的代码库开发者接受了这种设定后，他们开始大量依赖这种特性(即默认Python内部对象是thread-safe的，无需在实现时考虑额外的内存锁和同步操作)。
Python的线程虽然是真正的线程，但是由于GIL这把全局排他锁。这对多线程的效率有不小影响。甚至就几乎等于Python是个单线程的程序；GIL使得同一个时刻只有一个线程在一个CPU上执行字节码(GIL会根据执行的字节码行数,比如1000条字节码，以及根据时间片释放GIL，此外GIL在遇到IO的操作时候会主动释放)。 只有GIL主动得到释放，才能让别的线程有机会执行, 即多线程在Python中只能交替执行，无法将多个线程映射到多个CPU上执行, 即使100个线程跑在100核的CPU上，也只能用到1个核。所以，在CPU密集型应用中，Python多线程由于多线程调度开销反而效率更低，Python多线程比较适合IO操作比较多的场景，比如爬虫。
此外，通常我们用的Python解释器是官方实现的CPython。 在Python中，可以使用多线程，但不要指望能有效利用多核。如果一定要通过多线程利用多核，那只能通过C扩展来实现，或者重写一个不带GIL的解释器。
问题延申：多个线程共享全局变量时候，由于GIL在执行字节码切换时，导致异常，经典例子如下：
```python
# coding=utf-8
import threading

total = 0

def add():
    global total
    for i in range(1000000):
        total += 1

def desc():
    global total
    for i in range(1000000):
        total -= 1

thread1 = threading.Thread(target=add)
thread2 = threading.Thread(target=desc)
thread1.start()
thread2.start()

thread1.join()
thread2.join()
print(total)
```
上面这个例子每次运行的结果并不是我们期待的0，而是每次都不一样，这就需要后面会讲到的多线程同步问题。

## Python多线程编程：threading
Python可以通过实例化Thread类对象(threading.Thread(target=add))，或者通过继承Thread类来创建线程。 常用的接口有start、run、join和setDaemon，start用来开启一个线程，重写的run方法用来处理业务，join用来阻塞主线程，以等待子线程结束后再结束主线程,setDaemon用来设置一个线程是否为守护线程，如果被设置为守护线程，则主线程运行结束后，会直接kill掉子线程。
下面有一个请客吃火锅的例子，挺有意思的。
老王请小明和小王吃火锅，吃完火锅的时候会有以下三种场景：
场景一：老王先吃完了，小明和小王还没吃饱，老王不等他们俩，结账先走一步，剩下他们俩继续吃。
场景二：老王先吃完了，小明和小王还没吃饱，老王不等他们俩，结账一起走。
场景三：老王先吃完了，小明和小王还没吃饱，老王坐等他们俩, 最后结账一起走。

```python
# coding=utf-8
import threading
import time

def EatHuoPot(FriendName):
	print("%s:吃火锅开始" % FriendName)
	time.sleep(2)
	print("%s:吃火锅结束" % FriendName)

class MyThread(threading.Thread):
	def __init__(self, friend_name, thread_name):
		threading.Thread.__init__(self)
		self.m_friend_name = friend_name
		self.m_thread_name = thread_name

	def run(self):  # 把要执行的代码写到run函数里面 线程在创建后会直接运行run函数
		print("开始线程: " + self.m_thread_name)
		EatHuoPot(self.m_friend_name)  # 执行任务
		print("结束线程: " + self.m_thread_name)

print("老王请小伙伴开始吃火锅：！！！")
thread1 = MyThread("小明", "Thread-1")
thread2 = MyThread("小王", "Thread-2")

# 场景一：老王先吃完了，小明和小王还没吃饱，老王不等他们俩，结账先走一步，剩下他们俩继续吃。
# 开启线程
thread1.start()
thread2.start()

time.sleep(0.1)
print("退出主线程：老王吃火锅结束，结账走人")

# 执行结果：
'''
老王请小伙伴开始吃火锅：！！！
开始线程: Thread-1
小明:吃火锅开始
开始线程: Thread-2
小王:吃火锅开始
退出主线程：老王吃火锅结束，结账走人
小明:吃火锅结束
结束线程: Thread-1
小王:吃火锅结束
结束线程: Thread-2
'''

# 场景二：老王先吃完了，小明和小王还没吃饱，老王不等他们俩，结账一起走。
# 设置是否为守护线程,如果不设置,则默认为False; 如果设置为True,则主线程执行完后,不管子线程是否完成,一并和主线程退出
# 注意必须在start方法调用之前设置
thread1.setDaemon(True)
thread2.setDaemon(True)

thread1.start()
thread2.start()

time.sleep(0.1)
print("退出主线程：老王吃火锅结束，结账走人")

# 执行结果：
'''
老王请小伙伴开始吃火锅：！！！
开始线程: Thread-1
小明:吃火锅开始
开始线程: Thread-2
小王:吃火锅开始
退出主线程：老王吃火锅结束，结账走人
'''
# 场景三：老王先吃完了，小明和小王还没吃饱，老王坐等他们俩, 最后结账一起走。
# 开启线程
thread1.start()
thread2.start()

# 阻塞主线程，等子线程结束
thread1.join()
thread2.join()

time.sleep(0.1)
print("退出主线程：老王吃火锅结束，结账走人")

# 执行结果：
'''
老王请小伙伴开始吃火锅：！！！
开始线程: Thread-1
小明:吃火锅开始
开始线程: Thread-2
小王:吃火锅开始
小明:吃火锅结束
结束线程: Thread-1
小王:吃火锅结束
结束线程: Thread-2
退出主线程：老王吃火锅结束，结账走人
'''
```

## Python线程间通信
方法一：使用共享变量方式(操作共享变量是需要Lock锁)
```python
# coding=utf-8
# 简单模拟爬虫例子：通过共享变量detail_url_list来实现线程之间通信

import time
import threading

detail_url_list = []

def get_detail_html(lock):
	# 爬取文章详情页
	global detail_url_list
	while True:
		if len(detail_url_list):
			lock.acquire()
			if len(detail_url_list):
				url = detail_url_list.pop()
				lock.release()
				print("get detail html started")
				time.sleep(2)
				print("get detail html end")
			else:
				lock.release()
				time.sleep(1)


def get_detail_url(lock):
	# 爬取文章列表页
	global detail_url_list
	while True:
		print("get detail url started")
		time.sleep(2)
		for i in range(20):
			lock.acquire()
			if len(detail_url_list) >= 10:
				lock.release()
				time.sleep(1)
			else:
				detail_url_list.append("https://movie.douban.com/top250?start=25&filter=/{id}".format(id=i))
				lock.release()

		print("get detail url end")

if __name__ == "__main__":
	lock = threading.Lock()
	get_url_thread = threading.Thread(target=get_detail_url, args=(lock,)) #抓取文章url
	get_url_thread.start()

	for i in range(3):
		get_html_thread = threading.Thread(target=get_detail_html, args=(lock,)) #抓取文章详情
		get_html_thread.start()

```

方法二：通过Queue(使用了线程安全deque实现)的方式进行线程间同步
```python
# coding=utf-8

import time
import threading
from queue import Queue

def get_detail_html(detail_url_queue):
	# 爬取文章详情页
	while True:
		url = detail_url_queue.get()
		print("get detail html started")
		time.sleep(2)
		print("get detail html end")

def get_detail_url(detail_url_queue):
	# 爬取文章列表页
	while True:
		print("get detail url started")
		time.sleep(2)
		for i in range(20):
			detail_url_queue.put("https://movie.douban.com/top250?start=25&filter=/{id}".format(id=i))
		print("get detail url end")

if __name__ == "__main__":
	detail_url_queue = Queue(maxsize=1000)
	get_url_thread = threading.Thread(target=get_detail_url, args=(detail_url_queue,))
	get_url_thread.start()

	for i in range(3):
		get_html_thread = threading.Thread(target=get_detail_html, args=(detail_url_queue,))
		get_html_thread.start()
```

## Python线程间同步
#### 线程同步方式一：锁机制Lock、RLock
通过加锁对共享变量的访问进行控制使用，下面从字节码解析角度(假设a是全局共享变量)，分析Python解释器(根据执行字节码条数长度或者执行字节码时间片长度)在代码片段之间进行切换带来的问题。
```python
# coding=utf-8
import dis

def add(a):
	a += 1

def desc(a):
	a -= 1

print(dis.dis(add))
print(dis.dis(desc))

'''
22   0 LOAD_FAST           0 (a)
     2 LOAD_CONST          1 (1)
     4 INPLACE_ADD
     6 STORE_FAST          0 (a)
     8 LOAD_CONST          0 (None)
     10 RETURN_VALUE
None
 26  0 LOAD_FAST           0 (a)
     2 LOAD_CONST          1 (1)
     4 INPLACE_SUBTRACT
     6 STORE_FAST          0 (a)
     8 LOAD_CONST          0 (None)
     10 RETURN_VALUE
None
'''
下面简要分析一下：在第1步至第4步过程中，如果GIL发生切换，则不能保证a的值正常。
# add
'''
1. load a a=0
2. load 1 1
3. +    1
4. 赋值给a a=1
'''

# desc
'''
1. load a a=0
2. load 1 1
3. -    -1
4. 赋值给a a=-1
'''
```
所以，在对共享变量进行访问之前，需要先获取锁，在访问结束时，再释放锁
```python
# coding=utf-8

import threading
from threading import Lock, RLock
# Lock不能连续多次acquire，否则锁死；RLock为可重入的锁，可以连续多次acquire，但必须有相应次数的release 

total = 0
lock = Lock()

def add():
	global lock, total
	for i in range(1000000):
		# 获取锁
		lock.acquire()
		total += 1
		# 释放锁
		lock.release()

def desc():
	global lock, total
	for i in range(1000000):
		# 获取锁
		lock.acquire()
		total -= 1
		# 释放锁
		lock.release()

thread1 = threading.Thread(target=add)
thread2 = threading.Thread(target=desc)
thread1.start()
thread2.start()
thread1.join()
thread2.join()
print(total)
# 运行结果始终为0

```
锁的缺点：
1.用锁会影响性能，获取锁和释放锁都需要时间
2.锁会引起死锁：由于线程之间同时占有对方请求的资源,比如死锁的情况
<br/>A(a、b)<br/>acquire(a)<br/>acquire(b)</br>
<br/>B(a、b)<br/>acquire(a)<br/>acquire(b)</br>

#### 线程同步方式二：条件变量Condition
条件变量允许一个或多个线程进入到等待状态，直到它们被其他线程唤醒，用于复杂的线程间同步；Condition类最重要的两个接口wait和notify，wait用来等待线程被notify唤醒，条件变量最典型的应用就是生产者/消费者问题，下面以2个人对古诗为例进行说明。
```python
# coding=utf-8
import threading

class XiaoAi(threading.Thread):
    def __init__(self, cond):
        super().__init__(name="小爱")
        self.cond = cond

    def run(self):
		with self.cond:
			self.cond.wait()
			print("{} : 在 ".format(self.name))
			self.cond.notify()

			self.cond.wait()
			print("{} : 好啊 ".format(self.name))
			self.cond.notify()

			self.cond.wait()
			print("{} : 君住长江尾 ".format(self.name))
			self.cond.notify()

			self.cond.wait()
			print("{} : 共饮长江水 ".format(self.name))
			self.cond.notify()

			self.cond.wait()
			print("{} : 此恨何时已 ".format(self.name))
			self.cond.notify()

			self.cond.wait()
			print("{} : 定不负相思意 ".format(self.name))
			self.cond.notify()

class TianMao(threading.Thread):
    def __init__(self, cond):
        super().__init__(name="天猫精灵")
        self.cond = cond

    def run(self):
        with self.cond:
            print("{} : 小爱同学 ".format(self.name))
            self.cond.notify()
            self.cond.wait()

            print("{} : 我们来对古诗吧 ".format(self.name))
            self.cond.notify()
            self.cond.wait()

            print("{} : 我住长江头 ".format(self.name))
            self.cond.notify()
            self.cond.wait()

            print("{} : 日日思君不见君 ".format(self.name))
            self.cond.notify()
            self.cond.wait()

            print("{} : 此水几时休 ".format(self.name))
            self.cond.notify()
            self.cond.wait()

            print("{} : 只愿君心似我心 ".format(self.name))
            self.cond.notify()
            self.cond.wait()

if __name__ == "__main__":
    cond = threading.Condition()
    xiaoai = XiaoAi(cond)
    tianmao = TianMao(cond)

	xiaoai.start()
    tianmao.start()
	# 注意要点
    # 1.启动顺序很重要，一般情况下，先启动等待被唤醒的线程，再启动主动唤醒的线程
    # 2.在调用with cond之后才能调用wait或者notify方法，with cond相当于用acquire和release包住代码段
    # 3.Condition有两层锁， 底层锁会在线程调用了wait方法的时候释放，每次调用wait的时候会分配一把锁并放入到cond的等待队列中，等到notify方法的唤醒
执行结果：
天猫精灵 : 小爱同学 
小爱 : 在 
天猫精灵 : 我们来对古诗吧 
小爱 : 好啊 
天猫精灵 : 我住长江头 
小爱 : 君住长江尾 
天猫精灵 : 日日思君不见君 
小爱 : 共饮长江水 
天猫精灵 : 此水几时休 
小爱 : 此恨何时已 
天猫精灵 : 只愿君心似我心 
小爱 : 定不负相思意
```
Condition类wait和notify源码分析参见：http://timd.cn/python/threading/condition/

#### 线程同步方式三：信号量Semaphore
Semaphore可以控制并发的线程数量(Semaphore内部维护了一个条件变量和一个计数器)，在做爬虫的时候会经常遇到，有时候爬取速度太快了，会导致被网站禁止，所以这个时候就需要控制爬虫爬取网站的频率。下面以模拟爬虫的例子进行解析。
url_producer线程，爬取url，多个html_spider_thread线程，爬取url对应的网页。如果直接开20个htmlSpider线程，20个线程是同时执行的，现在要限制同时执行能执行三个，就可以使用信号量来控制。
```python
# coding=utf-8
import threading
import time

class HtmlSpider(threading.Thread):
	def __init__(self, url, sem):
		super().__init__()
		self.url = url
		self.sem = sem

	def run(self):
		time.sleep(2)
		print("got html text success")
		self.sem.release() #每当调用release()时，内置计数器+1，并让某个线程的acquire()从阻塞变为不阻塞


class UrlProducer(threading.Thread):
	def __init__(self, sem):
		super().__init__()
		self.sem = sem

	def run(self):
		for i in range(20):
			self.sem.acquire() # 每当调用acquire()时，内置计数器-1,直到为0的时候阻塞
			html_spider_thread = HtmlSpider("https://baidu.com/{}".format(i), self.sem)
			html_spider_thread.start()


if __name__ == "__main__":
	sem = threading.Semaphore(value = 3) #value是内部维护的计数器的大小，默认为1 
	url_producer = UrlProducer(sem)
	url_producer.start()
```

## Python线程池：concurrent.futures
虽然信号量可以当作一个粗略版本的线程池，但是主线程不方便获取某一个线程的状态，以及返回值。
下面以一个小例子进行解析：
初始化一个最大容量为 2 的进程池。然后我们调用进程池中的 submit 方法提交一个任务。好了有意思的点来了，我们在调用 submit 方法后，得到了一个特殊的变量，这个变量是 Future 类的实例，代表着一个在未来完成的操作。换句话说，当 submit 返回 Future 实例的时候，我们的任务可能还没有完成，我们可以通过调用 Future 实例中的 done 方法来获取当前任务的运行状态，如果任务结束后，我们可以通过 result 方法来获取返回的结果。如果在执行后续的逻辑时，我们因为一些原因想要取消任务时，我们可以通过调用 cancel 方法来取消当前的任务。

```python
# coding=utf-8
from concurrent.futures import ThreadPoolExecutor, as_completed

# 初始化一个最大容量为 2 的进程池
executor = ThreadPoolExecutor(max_workers=2)

# 通过submit函数提交执行的函数到线程池中, submit 是立即返回
task1 = executor.submit(get_html, (3))
task2 = executor.submit(get_html, (2))

# 调用submit方法后，得到了一个特殊的变量，这个变量是Future类的实例，代表着一个在未来完成的操作。即当submit返回Future类实例的时候，我们的任务可能还没有完成；我们可以通过调用Future实例中的done方法来获取当前任务的运行状态；如果任务结束后，可以通过result方法来获取返回的结果。如果在执行后续的逻辑时，如果想要取消任务时，可以通过调用cancel方法来取消该任务。

# Future类实例done方法，可以获取当前任务的运行状态
print(task1.done())

# Future类实例cancel方法，可以取消某线程，注意该线程一旦已经开始，就不能取消
print(task2.cancel())

# Future类实例result方法，可以获取task的执行结果，这个方法是阻塞的
print(task1.result())

# as_completed：已经完成的任务
time.sleep(4)
for future in as_completed(all_task):
    data = future.result()
    print("get {} page".format(data))

```

## Python多进程编程
### Python多线程和多进程对比
根据前面GIL的分析，针对耗CPU的操作，比如人工智能算法，挖矿计算，宜采用多进程编程， 而对于IO高的操作，宜使用多线程编程，因为进程切换代价要高于线程；多进程编程使用方式和多线程类似，下面各举一个例进行对比。

1.耗CPU操作
```python
# coding=utf-8
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from concurrent.futures import ProcessPoolExecutor

def fib(n):
	if n <= 2:
		return 1
	return fib(n - 1) + fib(n - 2)

if __name__ == "__main__":
	with ProcessPoolExecutor(3) as executor:
		all_task = [executor.submit(fib, (num)) for num in range(25, 40)]
		start_time = time.time()
		for future in as_completed(all_task):
			data = future.result()
			print("exe result: {}".format(data))

		print("last time is: {}".format(time.time() - start_time))

运行结果：last time is: 18.460657119750977

if __name__ == "__main__":
	with ThreadPoolExecutor(3) as executor:
		all_task = [executor.submit(fib, (num)) for num in range(25, 40)]
		start_time = time.time()
		for future in as_completed(all_task):
			data = future.result()
			print("exe result: {}".format(data))

		print("last time is: {}".format(time.time() - start_time))

运行结果：last time is: 36.931291580200195
```
2.IO高频操作
```python
# coding=utf-8
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from concurrent.futures import ProcessPoolExecutor

def random_sleep(n):
	time.sleep(n)
	return n


if __name__ == "__main__":
	# 多进程方式
	with ProcessPoolExecutor(3) as executor:
		all_task = [executor.submit(random_sleep, (num)) for num in [2] * 30]
		start_time = time.time()
		for future in as_completed(all_task):
			data = future.result()
			print("exe result: {}".format(data))

		print("last time is: {}".format(time.time() - start_time))

运行结果：last time is: 20.114237785339355

if __name__ == "__main__":
	# 多线程方式
	with ThreadPoolExecutor(3) as executor:
		all_task = [executor.submit(random_sleep, (num)) for num in [2] * 30]
		start_time = time.time()
		for future in as_completed(all_task):
			data = future.result()
			print("exe result: {}".format(data))

		print("last time is: {}".format(time.time() - start_time))

运行结果：last time is: 20.009515285491943
```

### Python多进程编程
1.fork()
<br/>①. fork作用是克隆进程，将原先的一个进程再克隆出一个进程来，克隆出的这个进程就是原进程的子进程，这个子进程和其他的进程没有什么区别，同样拥有自己的独立的地址空间。不同的是子进程是在fork返回之后才开始执行的，执行fork之后，父子进程就分道扬镳。
<br/>②. fork只能用于linux/unix中；
<br/>③. fork有三个返回值，该进程为父进程时，返回子进程的pid; 该进程为子进程时, 返回0; fork执行失败，则返回-1

```python
import os
import time
print("test!!!")
pid = os.fork()
if pid == 0:
  print('子进程 {} ，父进程是： {}.' .format(os.getpid(), os.getppid()))
else:
  print('我是父进程：{}.'.format(pid))

 time.sleep(2)

执行结果为：
test!!!
我是父进程：24795
子进程 24795 ，父进程是：24794 
```

2.multiprocessing模块
<br/>与创建线程类似，可以利用multiprocessing.Process创建一个进程。与Thread对象的用法相同，Process对象也有start(), run(), join()等方法。
```python
import time
import multiprocessing

def get_html(n):
	time.sleep(n)
	print("sub_progress success")
	return n

#使用Process创建进程
if __name__ == "__main__":
	progress = multiprocessing.Process(target=get_html, args=(2,))
	progress.start()
	print(progress.pid)
	progress.join()
	print("main progress end")
运行结果：
13040
sub_progress success
main progress end

#如果要启动大量的子进程，可以用进程池的方式批量创建子进程：
if __name__ == "__main__":
	pool = multiprocessing.Pool(multiprocessing.cpu_count())
	result = pool.apply_async(get_html, args=(3,))

	# 调用join()之前必须先调用close()，调用close()之后不能继续添加新的Process
	pool.close()
	
	#等待所有任务完成：
	pool.join()
	print(result.get())
运行结果：
sub_progress success
3
```

3.进程间通信：
<br/>多进程应该避免共享资源，在多线程中，我们可以比较容易地共享资源，比如使用全局变量或者传递参数。在多进程情况下，由于每个进程有自己独立的内存空间，以上方法并不合适。我们可以通过共享内存和Manager的方法来共享资源。但这样做提高了程序的复杂度，并因为同步的需要而降低了程序的效率。
<br/>①.与线程间通信不一样，进程之间的数据是相互独立隔离的，所以共享全局变量方式不适用于多进程之间的通信。
```python
import time
from multiprocessing import Process

def producer(a):
	a += 100
	time.sleep(2)

def consumer(a):
	time.sleep(2)
	print(a)


if __name__ == "__main__":
	a = 1
	my_producer = Process(target=producer, args=(a,))
	my_consumer = Process(target=consumer, args=(a,))
	my_producer.start()
	my_consumer.start()
	my_producer.join()
	my_consumer.join()
运行结果：1
```

<br/>②.multiprocessing中的queue不能用于pool进程池, pool中的进程间通信需要使用Manager中的Queue

```python
import time
from multiprocessing import Process, Queue, Pool, Manager, Pipe

def producer(queue):
	queue.put("a")
	time.sleep(2)

def consumer(queue):
	time.sleep(2)
	data = queue.get()
	print(data)

if __name__ == "__main__":
	queue = Manager().Queue(10)
	pool = Pool(2)

	pool.apply_async(producer, args=(queue,))
	pool.apply_async(consumer, args=(queue,))

	pool.close()
	pool.join()
运行结果：a
```
<br/>③.通过Pipe实现进程间通信，需要注意的是，Pipe只能适用于两个进程，由于Queue用于进程之间同步时，需要加很多锁,所以Pipe的性能要高于Queue。Pipe可以是单向(half-duplex)，也可以是双向(duplex)的。单向管道只允许管道一端的进程输入，而双向管道则允许从两端输入。Pipe对象建立的时候，默认为双向的，即返回两个元素，每个元素代表Pipe的一端(为Connection对象)。可以对Pipe的某一端调用send方法来传送对象，在另一端使用recv来接收。
```python
import time
from multiprocessing import Process, Queue, Pool, Manager, Pipe

def producer(pipe):
    pipe.send("test")

def consumer(pipe):
    print(pipe.recv())

if __name__ == "__main__":
    recevie_pipe, send_pipe = Pipe() # 默认为双向
    my_producer= Process(target=producer, args=(send_pipe, )) # 发送对象
    my_consumer = Process(target=consumer, args=(recevie_pipe,)) # 接收对象

    my_producer.start()
    my_consumer.start()
    my_producer.join()
	my_consumer.join()
运行结果：test
```

<br/>④.通过Manager中的dict实现进程间内存共享
```python
import time
from multiprocessing import Process, Queue, Pool, Manager, Pipe

def add_data(p_dict, key, value):
	p_dict[key] = value

if __name__ == "__main__":
    progress_dict = Manager().dict()
    first_progress = Process(target=add_data, args=(progress_dict, "test1", 1))
    second_progress = Process(target=add_data, args=(progress_dict, "test2", 2))

    first_progress.start()
    second_progress.start()
    first_progress.join()
    second_progress.join()

    print(progress_dict)
运行结果：{'test1': 1, 'test2': 2}
```





