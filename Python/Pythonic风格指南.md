## Pythonic风格指南

![](http://qn.tangyingkang.com/image/blog/python/pythonic.png)

### 什么是Pythonic？

__Pythonic__是python开发者对于代码风格的一种追求，用图片上的话来说就是：
> 这很python！

用赖勇浩大牛的话来概括是：
> Pythonic 就是以清晰、可读的惯用法应用Python 理念和数据结构
  
再看看Tim Peters对于python之禅的诗篇，打开你的python解释器，输入```import this```：

	>>> import this
	The Zen of Python, by Tim Peters

	Beautiful is better than ugly.
	Explicit is better than implicit.
	Simple is better than complex.
	Complex is better than complicated.
	Flat is better than nested.
	Sparse is better than dense.
	Readability counts.
	Special cases aren't special enough to break the rules.
	Although practicality beats purity.
	Errors should never pass silently.
	Unless explicitly silenced.
	In the face of ambiguity, refuse the temptation to guess.
	There should be one-- and preferably only one --obvious way to do it.
	Although that way may not be obvious at first unless you're Dutch.
	Now is better than never.
	Although never is often better than *right* now.
	If the implementation is hard to explain, it's a bad idea.
	If the implementation is easy to explain, it may be a good idea.
	Namespaces are one honking great idea -- let's do more of those!

其中的要点在于这几句：	
> 美胜于丑，显胜于隐，简胜于杂，杂胜于乱，平胜于陡，疏胜于密  
> 找到简单问题的一个方法，最好是唯一的方法（正确的解决之道）  
> 难以解释的实现，源自不好的主意；如果有非常棒的主意，它的实现肯定易于解释

以上可能有玄(zhuang)虚(bi)。  
结合过去几年的python经验，我总结了自己对于pythonic的几点理解：

- 遵守[PEP8风格指南](https://www.python.org/dev/peps/)
- 易于阅读（接近伪代码）
    - 变量命名不易引起混淆
        - 不要害怕过长的变量名
        - 下划线风格
    - 正确换行缩进
        - 采用4个空格进行缩进而不是用tab键
        - 适当添加空行
        - 避免过长的代码行（每行不超过80个字符） 
    - 充足的注释
    - 结构清晰
    - 一个模块/文件的代码行数不应该超过500行，再怎么样也不能超过1000行
- 文件开头总是申明utf8编码
- 尽量使用绝对引用，而不是相对引用    
- 函数设计原则
    - 尽量短小，嵌套层次不宜过深
    - 函数名正确反应功能
    - 参数设计简洁明了，容易猜出参数作用
    - 尽量考虑向下兼容（例如使用默认参数来包容版本升级）
    - 一个函数尽量只做一件事，保证语句粒度的一致性
- 类
    - 使用前缀下划线（假装）保护私有变量
    - 理解Mixin
- 常量集中在一个文件进行管理        
- 摒弃其他语言的风格，充分利用python自身特性，掌握python"惯用法"：
    - python装饰器（decorator）
    - python描述符（descriptor）
    - ```*``` ```**``` 参数解包（args unpacking）
    - 利用元祖特性交换变量
    - 列表切片
    - 列表解析（list comprehensions）和生成器表达式
    - 善用enumerate()进行遍历
    - 善用zip()方法（如创建dict）
    - 使用format()格式化字符串替换
    - 使用join()方法拼接字符串
    - 使用with语句管理上下文资源
    - 理解yield关键字，掌握生成器（generator）
    - 理解元类，（必要时）代码运行时动态地创建类
    - 通过重写内部方法来创建自定义类
    - 使用```if __name__ == '__main__'```来保持模块的引用行和独立性
    
- 熟练使用python高级数据结构和标准库，通读或经常查阅官方文档
    - collections.counter
    - collections.deque
    - collections.defaultdict
    - array.array
    - heapq
    - bisect
    - weakref
    - copy.copy/copy.deepcopy
    - itertools
    - functools
    - re
    - os/sys
    - subprocess
    - pdb/traceback/logging
    - threadding/multiprocessing
    - urllib/urllib2/httplib
    - Queue
    - pickle/cPickle/json
    - hashlib
    - timeit/cProfile
    - atexit
    - dis
- 熟悉社区里较为推崇的的第三方模块
    - requests
    - (太多了写不完...)


__另外还有...__  

- 一定的审美水准
- 轻微的代码洁癖
- 做个优雅的程序员

![](http://qn.tangyingkang.com/image/jgz02/22.gif)

