---
title: 探索 Python 函数对象参数默认值陷阱
date: 2018-08-23 22:43:14
---
本文纯属个人意淫,没有任何科学依据:

**现在有一下两个程序,那么他们打印的都是什么呢?**

```python
def dict_updater(k, v, dic={}):
    dic[k] = v
    print(dic)

dict_updater('one', 1)
dict_updater('two', 2)
dict_updater('three', 3, {})
dict_update('four',4)
```

```python
def demo(k, v):
    def dict_updater(k, v, dic={}):
   	dic[k] = v
    	print(dic)
    dict_updater(k, v)

demo('one', 1)
demo('two', 2)
demo('three', 3)
```

**为了弄清楚他们打印的是什么,让我们先来了解一下一下几个问题:**


### 1. 函数对象`dict_updater`在何时构造并初始化

* 如果函数所在文件是main文件,则在main执行到`dict_updater`所在行时初始化

* 在第一次 import 时进行

之后函数对象被关联到当前的命名空间  引用计数+1

### 2. 函数对象如何存储默认参数

默认值在函数对象创建时生成, 保存在 `__defaults__`, 为每次调用所共享.
当显示给默认参数赋值时,此时不再调用`__defaults__`的默认值,而是把默认变量名关联到新的传参对象.
但是并没有改变`__defaults__`的值,因为他是`tuple`

```python
def test(a, x=[]):
    x.append(a)
    print(x)
    print(test.__defaults__)

In [2]: test.__defaults__
Out[2]: ([],)

In [3]: type(test.__defaults__)
Out[3]: tuple

In [4]: test(1)
[1]
([1],)

In [5]: test(2)
[1, 2]
([1, 2],)

In [6]: test(3, [])
[3]
([1, 2],)

In [7]: test(4)
[1, 2, 4]
([1, 2, 4],)

```
关于`tuple`
```python
In [11]: l = []

In [12]: t = (l,)

In [13]: t
Out[13]: ([],)

In [14]: l.append(1)

In [15]: l
Out[15]: [1]

In [16]: t
Out[16]: ([1],)

In [17]: l.append(2)

In [18]: t
Out[18]: ([1, 2],)

In [19]: l = []

In [20]: t
Out[20]: ([1, 2],)

In [21]: t[0]
Out[21]: [1, 2]

In [22]: t[0] = []
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-22-33351fe06dab> in <module>()
----> 1 t[0] = []

TypeError: 'tuple' object does not support item assignment


```

### 3. 可变对象与不可变对象

1. 可变类型（列表、字典）

　　　　　　所谓可变对象是指，对象的内容可变，而不可变对象是指对象内容不可变。
```python
>>> aList = ['java',66,88,'python']
>>> aList
['java', 66, 88, 'python']
>>> aList[2]
88
>>> id(aList)  #注意观察id值
48255112L
>>> aList[2] = aList[2] + 12
>>> aList[3] = "python2"
>>> aList
['java', 66, 100, 'python2']
>>> id(aList)
48255112L　　#注意观察id值
```
2. 不可变类型（数字、字符串、元祖）

　　　　注意喽：下面的例子中，事实上是一个新对象被创建，然后它取代了旧对象。新创建的对象被关联到原来的变量名，旧对象被丢弃，垃圾回收器会在适当的时机回收这些对象。
```python
>>> v1 = "python"
>>> id(v1)
31631080L
>>> v1 = "java"
>>> id(v1)
31632240L  #由于str是不可变的，重新创建了java对象，随之id改变，旧对象python会在某个时刻被回收
>>>
>>> v2 = 12
>>> v3 = 12
>>> id(v2),id(v3)　　#同指同一内存区域，id相同
(31489840L, 31489840L)
>>> v2 += 1
>>> v2
13
>>> id(v2),id(v3)
(31489816L, 31489840L)
>>>

```

### 4. 函数对象的生命周期

函数算是 `function` 的实例。把 `def func()` 理解成 `func = new Function()` ，此时该函数实例计数是 1 ，如果此时删除其引用 `del func` 或者给 `func`赋值其他值，致使函数对象引用计数为0，那它就被回收了，同理的 `class Test(object)` 视为 `Test = new type()`，它的引用计数同一般普通的变量计算方式一样，没什么特别的。所以只要引用计数不为 0，那它的生命周期就是整个程序的生命周期。

如果是一个全局函数,那么不可避免的需要直接或间接被`main()`函数引用, 只有在main结束后引用计数才会归零,被垃圾回收.

如果是一个内部函数(不考虑高阶函数), 在外部函数执行完之后,该函数引用计数会被写为0,被垃圾回收.

我们通过观察进程内存验证一下：
![](/images/2018_08_23_02.png)
![](/images/2018_08_23_01.png)


### 解答：

好了，既然以上几个问题都弄懂了，自然也就知道打印什么了。
