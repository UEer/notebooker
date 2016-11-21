## Python装饰器


作者：[EaconTang](https://github.com/EaconTang) | 文章出处：[博客链接](http://blog.tangyingkang.com/post/2015/12/25/python-decorator/)  

----

![](http://qn.tangyingkang.com/image/blog/decorator.png)

Python的装饰器(Decorator)是个很优雅的语法糖，它让工程中的代码复用变得很艺术。  

### 装饰器大概是怎样的
装饰器的语法以"@"开头，类似于Java/C#的"@Annotation"，就是在定义的函数上一行加一个"@xxx"来为这个方法装饰一些功能。在某种程度上：**装饰器并不是一个功能特性，只是一个语法糖；因为python装饰器的本质是将函数或对象作为参数传递给另一个函数或对象，最后再返回新的函数对象，这是一个早就有的概念，装饰器只是这个旧概念的新语法**。  
比如说：

	@decorator
	def foo():
		pass

它实际上相当于：

	def foo():
		pass
		
	foo = decorator(foo)
  
再比如：

	# 装饰器可以多个叠加
	@decorator_1
	@decorator_2
	def foo():
		pass
		
相当于：

	foo = decorator_1(decorator_2(foo))	
		
再看一个简单的，在我的博文[《感受Python的编程特性》](/post/2015/08/03/python-features/)曾用过这个例子：

	# 这里定义一个装饰器，它在函数运行前后会分别print一些内容
	# 装饰器函数需要一个参数func，就是指被装饰的原函数 
	def foo_wrapper(func):
		def wrapper(*args, **kwargs):
			print 'start calling function... '	# 运行前
			print 'args: ', args, kwargs
			res = func(*args, **kwargs)		# 这一行运行被装饰的原函数
			print 'finished!'			# 运行后
			return res
		return wrapper
	
	@foo_wrapper
	def foo(a, b):
		print a + b

	foo(1, 2)
	# Out: 
	# start calling funtion...		# 这一行是装饰器输出的
	# args: [1, 2], {}				# 这一行是装饰器输出的
	# 3									# 这一行是原函数输出的
	# finished!						# 这一行是装饰器输出的
		
搞过Java的人都撸过设计模式，设计模式里面有个装饰器模式，听起来和python装饰器很相似，**两者都是对一个已有的功能对象做一些“修饰工作”（这些修饰一般是指可复用的小功能），同时又尽量不表现得太有侵入性**。  
但相比之下，Java的装饰器模式就显得厚重啰嗦，而Python装饰器则优雅敏捷，不需要纠结复杂的OO模型，完全就是一种函数式编程的快感！就像这样：  
![](http://qn.tangyingkang.com/image/bm01/01.gif)
  
  	
### 装饰器的装饰器
装饰器在调用时，还可以再传入参数来控制装饰行为，实际上，带参数的装饰器有更灵活的应用场景和价值。看这个例子：

	# 这个装饰器把会原函数重复调用两次
	def repeat_twice(func):
		def wrapper(*args, **kwargs):
			func(*args, **kwargs)
			func(*args, **kwargs)
		return wrapper
		
	@repeat_twice
	def foo():
		print 'hello world'
		
	foo()
	# Out:
	# hello world
	# hello world
	
考虑一下，如果我们需要把这个装饰器修改为重复调用原函数3次、4次...n次呢？总不能每次都修改装饰器吧？这时候，我们可以定义一个**装饰器的装饰器**，让装饰器在调用的时候可以传入一个参数n，使得原函数可以调用n次：

	def repeat(n):
		def repeat_n(func):
			def wrapper(*args, **kwargs):
				for i in range(n):
					# 这里循环n次调用原函数
					func(*args, **kwargs)
			return wrapper
		return repeat_n
		
	@repeat(3)
	def foo():
		print 'hello world'
		
	foo()
	# Out:
	# hello wolrd
	# hello wolrd
	# hello wolrd
		
哈哈，成功了吧？这里，我们可以把repeat()函数看成是一个装饰器工厂，repeat_n()函数是工厂创建的装饰器，wrapper()函数是最终执行返回的函数。


### 装饰器的副作用
到这里，如果你已经明白了装饰器的原理，你可能会意识到一个问题：
> 既然被装饰的函数已经是新的函数，那它的元属性岂不是也不见了？   

没错，如果我们对上述例子进行检查，会发现被装饰过的foo函数，它的元属性都变成了wrapper函数(装饰器最终返回的那个函数)的信息。比如`foo.__name__`不再是'foo'，而是'wrapper'。  
这个问题在实际工程中，可能会是个坑。所以python标准库的functools包提供了一个解决办法，用来消除这个副作用。我们可以这样改写上面的例子：

	from functools import wraps	# 引入functools的wrap装饰器方法
	
	def repeat_twice(func):
		@wraps(func)
		def wrapper(*args, **kwargs):
			func(*args, **kwargs)
			func(*args, **kwargs)
		return wrapper
		
	@repeat_twice
	def foo():
		print 'hello world'
		
	print foo.__name__		# Out: foo


### 装饰器的应用场景
总结完装饰器的内容，我们再看看装饰器有哪些应用场景。  
[Python Decorator Library](http://wiki.python.org/moin/PythonDecoratorLibrary) 是Pytho官方的wiki网站收集的装饰器的用法示例，可以作为参考。一般情况下，装饰器主要可以用在缓存、日志、URL路由和权限校验等常见的功能上，下面简单举几个例子。  
#### 缓存
假设有一个函数的运行时间比较长，为了节省时间，我们可以对其结果进行缓存，代码如下：

	import time
	from functools import wraps
	
	cache = {}		# 用于缓存函数结果的全局字典
	
	# 这个装饰器可以对原函数计算结果进行缓存，并可以通过参数duration指定缓存的时间
	def memorize(duration)
		def _memorize(func):
			@wraps(func)	
			def wrapper(*args, **kwargs):
				# 将函数名和参数拼接为字符串作为缓存的key
				key = '{}/{}/{}'.format(func.__name__, args, kwargs)
				# 如果有这个key，说明缓存过
				if key in cache:
					# 检查这个缓存结果有无超时失效
					if time.time() - cache[key]['time'] <= duration:
					return cache[key]['result']
				# 缓存无效，那么执行原函数，并把结果存入缓存字典
				res = func(*args, **kwargs)
				cache[key] = {
					'result': res,
					'time': time.time()
				}
				return res
			return wrapper
		return _memorize
		
	@memorize(60)
	def foo(a):
		time.sleep(int(a))
		return a
		
	print foo(10)		# 10秒后才输出一个‘10’
	print foo(10)		# 再次运行时，由于已经缓存好，可以立即输出
	time.sleep(60)	# 间隔60秒时间，让缓存超时失效
	print foo(10)		# 由于缓存失效，所以这里又得等10秒才能输出


#### 单例模式的优雅实现
单例模式是简单经典的一个设计模式，我们看看用Python装饰器是怎样实现的：

	def singleton(cls):
		_instances = {}		# 容器字典，存储每个类对应生成的实例
		def wrapper(*args, **kwargs):
			# 检查类是否_instances的key；如果是，直接返回生成过的实例
			if cls not in _instances:
				_instances[cls] = cls(*args, **kwargs)
			return _instances[cls]
		return wrapper
		
	@singleton
	class Foo(object):
		def __init__(self, a=0):
			self.a = a
			
	foo1 = Foo(1)
	foo2 = Foo(2)
	print foo1.a			# Out: 1
	print foo2.a			# Out: 1
	print id(foo1) == id(foo2)		# 结果为True，两个实例id相等，证明这个类确实是单例的
