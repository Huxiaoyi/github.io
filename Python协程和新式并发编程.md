
# Python协程和新式并发编程
作为一名python拥趸者，如果不会python协程，不了解python协程的衍化历程，那么你掌握的python早就过时了，你的编程方式也早就过时了。

不可否认，曾经我们都习惯同步编程方式：在一个函数里面直接调用其它函数，遇到耗时的IO时，原地阻塞，直到阻塞函数返回数据，才继续往下执行。

同步编程的好处是思路自然、逻辑简洁不复杂，上下文维护也非常方便；缺点是对于IO密集型程序，会浪费大量昂贵的CPU时钟周期来等待阻塞接口返回数据，这会导致程序的并发性很低。
对于高并发需求程序，这是致命的缺点。 

所以我们开始学习异步编程，其实不论是注册回调方式的异步编程，还是利用python多线程，在一定程度上缓解了高并发的压力；但是对于层层嵌套的异步回调、回调过程中上下文的维护有时挺恶心的； 而对于多线程，虽然占用的资源相对于多进程线程少很多，但是对于高并发程序，成千上万的线程对系统资源开销还是难以承受的，而且多线程对于共享资源访问时容易造成灾难性的死锁问题； 此外多线程的切换调度由操作系统完成，这导致多线程程序往往不可控，无法通过API取消多线程等等。

本文讨论的主题-协程和新式异步编程，就是为了解决异步回调编程和多线程编程的缺点，以及来自其它深谙高并发的新式语言比如Go的压迫，通过python多个版本逐步衍化提出的。

协程，即协作式编程，又称作微线程，英文名Coroutine。我们知道子程序，也就是我们常说的函数，在所有语言中都是层级调用关系； 子程序调用通过栈实现，通常一个线程就是执行一个子程序。 子程序调用总是一个入口，执行完毕后一次返回，调用顺序是明确的。 而协程的调用和子程序不同，协程在执行过程中，可以在子程序内部暂停，并且在适当的时候再返回来继续执行
(通常在调用阻塞IO子程序处暂停执行，当阻塞IO子程序返回数据时，在暂停处恢复继续执行)，从而实现用同步方式编写异步程序。

由于协程是一个线程执行，协程的子程序切换不是线程切换，而是由程序自身控制，所以协程没有多线程开销，也没有线程切换的开销； 在协程中控制共享资源也不需要加锁，所以执行效率会比多线程高很多，并发性会更高；

相比多线程，协程底层实现更加复杂，Python协程是通过生成器实现的，并经历多个版本的改进，所以直接接触协程会比较晦涩，借用深入浅出MFC大作里的一句名言，勿在浮砂筑高台，首先深入理解python迭代器、生成器，然后了解用生成器实现的协程和原生的协程，从而一步步理解协程。

## 一. 可迭代的对象(Iterable)、迭代器(Iterator)
1.迭代是数据处理的基石，我们知道所有的序列都可以迭代，如果给定一个list或tuple，我们可以通过for循环来遍历这个list或tuple，这种遍历我们称为迭代(Iteration)。

在Python中，迭代是通过for ... in来完成的；相比于其他语言，python的for循环抽象程度更高：一切可以迭代的对象都可以用for循环进行迭代。

通常检查一个对象是否可以迭代，可以直接用isinstance(obj, Iterable)判断； 但从Python3.4开始，检查对象x能否迭代，最准确的方法是调用iter(x)函数。这比使用isinstance(x, Iterable)更准确，
因为iter(x)函数会考虑到遗留的__getitem__方法，而Iterable类则不考虑。这也是为什么早期python版本的字符串没有__iter__方法，却可以迭代的原因。

任何Python序列都可迭代的原因是，它们都实现了__getitem__方法。其实标准的序列也都实现了__iter__方法，之所以对__getitem__方法做特殊处理，是为了向后兼容。下面示例定义一个可迭代的类：
```python
# encoding: utf-8

from collections import Iterable


class Words(object):
	def __init__(self, words):
		self.m_words = words

	# 当for发现没有__iter__但是有__getitem__的时候，会从0开始依次读取相应的下标，直到发生IndexError为止，这是一种旧的迭代协议
	# Python语言内部会处理for循环和其迭代上下文(如列表推导、元组拆包等)中的StopIteration异常
	def __getitem__(self, index):
		return self.m_words[index]

words = Words(["life", "is", "short", "I", "use", "python"])

print "words is Iterable1?", isinstance(words, Iterable)	#False，该检查方法不准确
print "words is Iterable2?", isinstance(iter(words), Iterable)	#True

#直接遍历words对象
for word in words:
	print "for-word:", word

#words也是序列，所以也可以按索引获word
print "words[0-2]", words[0], words[1], words[2]

#使用iter()函数生成迭代器
for word in iter(words):
	print "iter-word:", word

输出：
words is Iterable1? False
words is Iterable2? True
for-word life
for-word is
for-word short
for-word I
for-word use
for-word python
words[0-2] life is short
iter-word life
iter-word is
iter-word short
iter-word I
iter-word use
iter-word python
```

下面说明python序列可以迭代具体的原因。序列可以迭代的原因：iter函数，解释器需要迭代对象x时，会自动调用iter(x)。
内置的iter函数有以下作用：
<br/>①. 检查对象是否实现了__iter__方法，如果实现了就调用它，获取一个迭代器。
<br/>②. 如果没有实现__iter__方法，但是实现了__getitem__方法，Python会创建一个迭代器，尝试按顺序(从索引0开始)获取元素。
<br/>③. 如果上述尝试都失败，则Python抛出TypeError异常，提示"TypeError: 'X' object is not iterable"(X对象不可迭代)，其中X是目标对象所属的类。

其实python的for循环本质上就是通过不断调用next()函数实现：
```python
# 通过iter()获得Iterator对象:
it = iter([1, 2, 3, 4, 5])
while True:
    try:
        x = next(it)
    except StopIteration:
        break
```

2.可迭代的对象和迭代器之间的关系
<br/>迭代器是访问集合内元素的一种方式，一般用来遍历数据；Python从可迭代的对象中获取迭代器, 迭代器用于从集合中取出元素(迭代器设计模式)。标准的迭代器继承自可迭代类，有两个魔法方法：
<br/>__next__：返回下一个可用的元素，如果没有元素了，抛出 StopIteration异常。
<br/>__iter__：返回self，以便在应该使用可迭代对象的地方使用迭代器，例如在for循环中。
<br/>由此可知，迭代器可以迭代，但是可迭代的对象不是迭代器(没有__next__)； 迭代器对象表示的是一个数据流，它可以被next()函数调用，并不断返回下一个数据，直到没有数据时抛出StopIteration错误。 此外，我们可以把这个数据流看做是一个有序序列，但我们却不能提前知道序列的长度，只能不断通过next()函数实现按需计算下一个数据，即Iterator的计算是惰性的，只有在需要返回下一个数据时它才会计算。 以下是迭代器版本示例：
```python
# encoding: utf-8

# 典型的迭代器类:可以不继承迭代器类，但必须实现__next__和__iter__接口
class WordIterator(object):
	def __init__(self, words):
		self.m_words = words
		self.m_index = 0

	# python3.4以上版本改为用__next__魔法函数
	def next(self):
		try:
			word = self.m_words[self.m_index]
		except IndexError:
			raise StopIteration()

		self.m_index += 1
		return word

	def __iter__(self):
		return self


# 可迭代类:必须实现__iter__接口
class Words(object):
	def __init__(self, words):
		self.m_words = words

	def __iter__(self):
		return WordIterator(self.m_words)	#返回一个迭代器


words = Words(["life", "is", "short", "I", "use", "python"])
for word in words:
	print "for-word", word

输出：
for-word life
for-word is
for-word short
for-word I
for-word use
for-word python
```

迭代器获取元素方式：next方法和for循环
```python
①. 使用next方法：每次调用next，输出迭代器的下一个元素，元素迭代结束，抛出StopIteration异常
>>> L=[1,2]
>>> it = iter(L)
>>> it
<listiterator object at 0x000000000380FAC8>
>>> print(next(it))
1
>>> print(next(it))
2
>>> print(next(it))

Traceback (most recent call last):
  File "<pyshell#16>", line 1, in <module>
    print (next(it))
StopIteration
>>> 

②. 使用for循环进行遍历：for循环内部会处理StopIteration异常
>>> it = iter(L)
>>> for x in it:
		print(x)

1
2
```

总结：凡是可作用于for循环的对象都是Iterable类型；凡是可作用于next()函数的对象都是Iterator类型，它们表示一个惰性计算的序列； list、dict、str等是Iterable但不是Iterator，不过可以通过iter()函数获得一个Iterator对象。

## 二. 生成器(generator)
1.生成器函数
<br/>由上可知，对于可迭代对象，无论是借助__getitem__，还是通过__iter__，目的都是返回一个迭代器； 实际上，迭代器的功能可以通过生成器函数来实现，从而免去单独定义一个迭代器类，示例如下：
```python
# encoding: utf-8

# 通过生成器函数代替迭代器功能
class Words(object):
	def __init__(self, words):
		self.m_words = words

	#凡是含有yield关键字的函数，都称为生成器函数，调用生成器函数时，会构建一个实现了迭代器接口的生成器对象
	def __iter__(self):
		for word in self.m_words:	#迭代self.words
			yield word				#产出当前word


words = Words(["life", "is", "short", "I", "use", "python"])
for word in words:
	print "for-word", word

输出：
for-word life
for-word is
for-word short
for-word I
for-word use
for-word python
```
生成器函数:只要python函数的定义体中有yield关键字，该函数就是生成器函数；调用生成器函数时，会返回一个生成器对象。即生成器函数是生成器工厂。

2.生成器
<br/>通过列表生成式，我们可以直接创建一个列表，即直接返回列表所有元素。由于内存限制，列表容量肯定是有限的。 尤其仅仅需要访问列表前面几个元素时，后面元素会白白占用很多空间。
<br/>所以，如果列表元素可以按照某种算法推算出来，那我们是否可以在循环的过程中不断推算出后续的元素呢？ 这样就不必创建完整的list，从而节省大量的空间。在Python中，这种一边循环一边计算的机制，称为生成器。 所以生成器也常用来读取大型数据，防止一次性加载受内存限制。
<br/>要创建一个生成器，有很多种方法。最简单的方式是使用生成器表达式，只要把一个列表生成式的[]改成()，就创建了一个generator：
```python
>>> L = [x * x for x in range(5)]
>>> L
[0, 1, 4, 9, 16]
>>> g = (x * x for x in range(5))
>>> g
<generator object <genexpr> at 0x0000000003601120>

即创建L和g的区别仅在于最外层的[]和()，L是一个列表，而g是一个generator。
```
此外由1知，可以通过生成器函数创建生成器，当调用生成器函数时，就会返回一个生成器。

3.生成器运行机制
<br/>生成器与迭代器区别：python迭代器协议定义了两个魔法方法：_next__和__iter__。 生成器对象实现了这两个方法，因此从这方面来看，所有生成器都是迭代器，可迭代。
<br/>当我们使用生成器表达式或者调用生成器函数时，仅仅返回一个生成器对象(该对象属于内建types.GeneratorType类型)； 对于生成器表达式，它只是提供一个推算算法；对于生成器函数，只有开始对生成器进行迭代时，才会开始执行生成器函数。
<br/>通常我们使用for循环进行迭代，for循环每次迭代时会隐式调用next(x)，前进到生成器函数中的下一个yield语句，暂停函数的执行，并产出yield右侧表达式计算出的值，返回给调用方。
```python
>>> def gen_test():
	print("gen start")
	yield 1
	print("gen con")
	yield 1+2
	print("gen end")


>>> gen = gen_test()	#调用生成器函数，返回一个生成器对象，没有任何输出内容，说明生成器函数并未执行
>>> gen
<generator object gen_test at 0x0000000003324CF0>
>>> next(gen)	#使用next迭代，运行到下一个yield处，函数暂停运行，产出值1，并返回给调用方
gen start
1
>>> next(gen)
gen con
3
>>> next(gen)	#后面已经没有yield，生成器终止，抛出StopIteration异常
gen end

Traceback (most recent call last):
  File "<pyshell#11>", line 1, in <module>
    next(gen)
StopIteration
>>> 
```

## 三. 生成器如何进化成协程-Coroutines via Enhanced Generators
协程的底层架构在"PEP 342—Coroutines via Enhanced Generators"(https://www.python.org/dev/peps/pep-0342/)中定义，并在Python2.5版本(2006年)实现；自此之后，yield关键字便可以在表达式中使用，而且新增了.send(value)、.throw(...)和.close()方法。 即生成器的调用方可以使用.send(...)方法发送数据给生成器函数，发送的数据会成为生成器函数中yield表达式的值；
.throw(...)可以让调用方抛出异常，在生成器中处理；.close()用于终止生成器。

此外，Python3.3版本(2012年)实现了"PEP 380—Syntax for Delegating to a Subgenerator"(https://www.python.org/dev/peps/pep-0380/)。 PEP 380对生成器函数的句法做了两处改动，以便更好地作为协程使用：
<br/>生成器可以返回一个值，以前如果在生成器中给return语句提供值，会抛出SyntaxError异常。
<br/>新引入yield from句法，使用它可以把复杂的生成器重构成小型的嵌套生成器。
<br/>1.python协程前生版本1：Python2.5版本协程(通过生成器函数实现协程)
```python
>>> def simple_coro(a):
	print('-> Started: a =', a)
	b = yield a
	print('-> Received: b =', b)
	c = yield a + b
	print('-> Received: c =', c)


# 创建协程，协程目前处于GEN_CREATED状态(即协程未启动)
>>> coro = simple_coro(1)

# 使用next(coro)或者coro.send(None)预激协程：向前执行协程到第一个yield表达式；
# 产出a的值(如果yield右侧没有表达式，则产出None,即不产出值)，并且暂停，等待下一步为b赋值；协程目前处于GEN_SUSPENDED状态，即协程在yield表达式处暂停
>>> coro.send(None)
('-> Started: a =', 1)
1

# 把2发给暂停的协程，赋值给b，并驱动协程继续向前执行到下一个yield处，并产出a+b的值3，协程再次暂停，等待下一步为c赋值
>>> coro.send(2)
('-> Received: b =', 2)
3

# 把3发给暂停的协程；赋值给c；并驱动协程继续向前执行到下一个yield处，由于没有yield，协程终止(处于GEN_CLOSED 状态，即协程执行结束)，导致生成器对象抛出StopIteration异常。
>>> coro.send(3)
('-> Received: c =', 3)

Traceback (most recent call last):
  File "<pyshell#11>", line 1, in <module>
    coro.send(3)
StopIteration
>>> 
```

对于协程内出现的异常，如果没有捕获到，协程会终止。如果试图重新激活协程，会抛出StopIteration异常。 可以通过generator.throw(exc_type[, exc_value[, traceback]])方式，显示把异常类型发给协程：这会使得生成器在暂停的yield表达式处抛出指定的异常。如果生成器处理了抛出的异常，代码会向前执行到下一个yield表达式，而产出的值会成为调用generator.throw方法得到的返回值。如果生成器没有处理抛出的异常，异常会向上冒泡，传到调用方的上下文中。

还可以通过generator.close()终止协程：这会使得生成器在暂停的yield表达式处抛出GeneratorExit异常。 如果生成器没有处理这个异常，或者抛出了StopIteration异常(通常是指运行到结尾)，调用方不会报错。 如果收到GeneratorExit异常，生成器一定不能产出值，否则解释器会抛出RuntimeError异常。

2.python协程前生版本2：Python3.3版本协程
<br/>(1). python3.3之前，如果生成器返回值，解释器会报句法错误。 python3.3版本以后，生成器支持返回值：异常对象的value属性保存着返回的值，继而可以通过return表达式传给调用方：
```python
>>> def coro_fib():
		n, a, b = 0, 0, 1
		fib_result = []
		while True:
			fib_result.append(b)	#保存斐波那契数列项
			res = yield b			#产出斐波那契数列项，并可以给协程发送None，终止for循环和协程
			if res is None:
				break

			a, b = b, a + b
			n = n + 1

		return fib_result			#返回生成的斐波那契数列列表项


>>> fib_core = coro_fib()
>>> next(fib_core)
1
>>> print(fib_core.send(1))
1
>>> print(fib_core.send(1))
2
>>> print(fib_core.send(1))
3
>>> print(fib_core.send(1))
5
>>> print(fib_core.send(1))
8
>>> print(fib_core.send(1))
13
>>> try:
		print(fib_core.send(None))
	except StopIteration as exc:
		result = exc.value				# 异常对象的value属性保存着返回的值，可以通过return表达式传给调用方

>>> result
[1, 1, 2, 3, 5, 8, 13]	
```

(2). python3.3新引入yield from句法：解释器不仅会捕获StopIteration 异常，还会把value属性的值变成yield from表达式的值。 相比于yield， yield from是全新的语言结构。它的作用比yield多很多：
<br/>①. yield from可用于简化for循环中的yield表达式：yield from x表达式对x对象所做的第一件事是，调用iter(x)，从中获取迭代器： 所以下面函数功能等效：
```python
>>> def chain1(*iterables):
		for it in iterables:
			for ele in it:
				yield ele

>>> def chain2(*iterables):
		for it in iterables:
			yield from it
>>> list(chain1(a, b))
[1, 2, 'a', 'b']
>>> list(chain2(a, b))
[1, 2, 'a', 'b']
```

②. Syntax for Delegating to a Subgenerator：通过yield from把职责委托给子生成器，这是yield from最重要的性质。

在生成器gen中使用yield from subgen()时，子生成器subgen会获得控制权，把产出的值传给gen的调用方，即调用方可以直接控subgen；与此同时，gen会阻塞，等待subgen终止。 通过yield from，可以把最外层的调用方与最内层的子生成器连接起来，中间的委派生成器会在yield from表达式处暂停，然后调用方可以直接把数据发给子生成器，子生成器最终通过return，再把产出的值发给调用方。子生成器返回之后，解释器会抛出StopIteration异常，并把返回值附加到异常对象上，此时委派生成器会恢复执行。

3.python协程前生版本3：Python3.4版本协程
<br/>为了凸显协程与普通函数区别，从python3.4版本起，直接用@asyncio.coroutine装饰器把一个生成器函数标记为coroutine类型。

4.python协程前生版本4：Python3.5版本协程(原生协程)
<br/>python3.4用asyncio提供的@asyncio.coroutine装饰器把一个生成器函数标记为coroutine类型，然后在coroutine内部用yield from调用另一个coroutine实现异步操作。
<br/>python为了使协程语义变得更加明确，以区别生成器，并更好地标识异步IO；和其他语言的异步IO一样，python3.5版本也引入了async和await关键词，用于定义原生协程，从而让协程的代码更简洁易读： async和await是针对coroutine的新语法，要使用新的语法，只需要做两步简单的替换：
<br/>①. 把@asyncio.coroutine替换为async；
<br/>②. 把yield from替换为await。
需要注意的是，python原生协程的定义是为了使协程语义变得更加明确，以区别生成器，所以在以async定义的函数里面不能有yield和yield from。


## 四. 使用asyncio处理高并发
从python3.4版本起，python标准库引入asyncio包，asyncio包是python非常重要的一个包，该包内置了对异步IO支持，
提供了一种全新的编写并发代码方式，摒弃以前使用多线程实现并发方式。
asyncio包使用事件循环驱动协程实现并发。简单来说，asyncio编程模型就是一个消息循环：在单线程里维护多个任务间的切换，以实现多任务的调度。
可以通过asyncio.get_event_loop()直接获取一个EventLoop的引用，然后把需要执行的协程扔到EventLoop中(通过yield from把控制权交还给事件循环以防阻塞主循环，从而实现并发)，
事件循环根据之前排定的协程开始执行；在阻塞的操作执行完毕后获得通知，获得通知后，主循环把结果发给暂停的协程，从而实现异步IO。
<br/>(1). asyncio包基础-Future(期物)
python3.4起，标准库中有两个名为Future的类：concurrent.futures.Future和asyncio.Future。
这两个类的作用相同：两个Future类的实例都表示可能已经完成，或者尚未完成的延迟计算。

