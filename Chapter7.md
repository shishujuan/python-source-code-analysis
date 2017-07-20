# Python源码剖析笔记7-类机制
> 拖了好一段时间了，终于有空来看看python中的类机制了。内容太多，感觉有些地方还是模糊的，先写一些吧，有错误烦请指出。

## 1 Python对象模型
### 1.1 概述
python2.2之前的这里就不考虑了，从2.2之后python对象分为两类，```class对象和instance对象```,另外还有个术语type用来表示```“类型”```，当然class有时候也表示类型这个概念，比如下面的代码，我们定义了一个名为A的class对象，它的类型是type。并且定义了一个实例对象a，它的类型是A。

```
class A(object):
    pass
a = A()

#测试代码
In [7]: a.__class__
Out[7]: __main__.A

In [8]: type(a)
Out[8]: __main__.A

In [9]: A.__class__
Out[9]: type

In [10]: object.__class__
Out[10]: type

In [12]: A.__bases__
Out[12]: (object,)

In [14]: object.__bases__
Out[14]: ()

In [15]: a.__bases__
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
<ipython-input-15-d614806ca736> in <module>()
----> 1 a.__bases__

AttributeError: 'A' object has no attribute '__bases__'

In [16]: isinstance(a, A)
Out[16]: True

In [17]: isinstance(A, object)
Out[17]: True

In [18]: issubclass(A, object)
Out[18]: True
```
### 1.2 Python对象之间关系
如1.1中看到的，我这里将 <type 'type'>这个特殊的class对象单独列出来，因为它很特别，是所有class对象的类型，这里我们称之为metaclass。而<type 'object'>则是所有对象的基类。它们两者之间还有联系，我们按照```is-kind-of```和```is-instance-of```来划分关系，所有class对象的type都是metaclass对象，即在Python的C实现中对应PyType_Type，即所有class对象都是<type 'type'>的实例(is-instance-of)。而所有class对象的直接或间接基类都是object，即对应Python的C实现中PyBaseObject_Type(is-kind-of)，更加具体的关系参见下图。

![对象之间的关系.png](http://upload-images.jianshu.io/upload_images/286774-3ba957cce94e713c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2 class对象和instance对象
### 2.1 slot和descriptor
Python中的class对象都是PyTypeObject结构体类型变量，比如type对应在C实现中是PyType_Type，int对应则是PyInt_Type。int的类型是type，但是比较特殊的type，它的类型是自己，如下所示。当然它们的基类都是object。

```
In [2]: type.__class__
Out[2]: type

In [3]: int.__class__
Out[3]: type

In [4]: int.__base__
Out[4]: object

In [5]: type.__base__
Out[5]: object
```

Python在初始化class对象时会填充tp_dict，这个tp_dict会用来搜索类的方法和属性等。Python会对class对象的一些特殊方法进行特殊处理，这就引出了slot和descriptor的概念，其中对于一些特殊方法比如```__repr__```，python中会设置一个对应的slot，由于slot本身不是PyObject类型的，所以呢会增加一个封装，也就是descriptor了，最终在一个class对象的tp_dict中，方法名如```__repr__```会指向一个descriptor对象，而descriptor对象是对slot的封装，slot中会有一个slot function，比如对应```__repr__```的就是slot_tp_repr方法，```__init__```指向的是slot_tp_init方法。这样，如果在一个class中重新定义了```__repr__```方法，则在创建class对象的时候，就会将默认的tp_repr指向的方法替换为该slot_to_repr方法，最终在执行tp_repr时，其实就是执行的slot_to_repr方法，而在slot_to_repr方法中就会搜索并找到该class对象中定义的```__repr__```方法并调用，这样就完成了方法的复写。

比如下面的代码中class A继承自list，如果没有复写```__repr__```,则在输出的时候会调用```list_repr```方法，打印的是```'[]'```,如果如下面这样复写了，则打印的是```'Python'```。

```
>> class A(list):
    def __repr__(self):
        return 'Python'
>> s = '%s' % A()
>> s
   'Python'
```

### 2.2 MRO简析
MRO是指python中的属性解析顺序，因为Python不像Java，Python支持多继承，所以需要设置解析属性的顺序。MRO搜索规则如下:

- 1)先从当前class出发，比如下面就是先获取D，发现D的mro列表tp_mro没有D，则放入D。
- 2)获得C，D的mro列表没有C，则加入C。此时，Python虚拟机发现C中存在mro列表，于是转而访问C的mro列表:
 	- 2.1)获得A，D的列mro表没有A，则加入A。
 	- 2.2)获得list，尽管D的mro列表没有list，但是后面B的mro列表里面有list，于是这里不把list放到D的mro列表，推迟到处理B时放入。
  	- 2.3)获得object，同理也推迟再放。
- 3)获得B，D的mro列表没有B，则放入B。转而访问B的mro列表:
  	- 3.1)获得list，将list放入D的mro列表。
  	- 3.2)获得object，将object放入D的mro列表。
- 4)最终，D的mro列表为(D,C,A,B,list,object)。可以打印```D.__mro__```查看。所以最终输出为```A:show```.

```
class A(list):
  def show(self):
    print 'A:show'

class B(list):
  def show(self):
    print 'B:show'

class C(A):
  pass

class D(C, B):
  pass


d = D()
d.show()
```

### 2.3 class对象和instance对象的```__dict__```
观察class对象和instance对象的__dict__，如下代码可以看到结果，class对象的__dict__对应的类的属性，而instance对象的__dict__则是存储的实例变量。

```
class A(object):
  a = 1
  b = 2

  def __init__(self):
    self.c = 3
    self.d = 4

  def test(self):
    pass

  def __repr__(self):
    return 'A'

a = A()
print A.__dict__
print a.__dict__
print a

##输出结果
{'a': 1, '__dict__': <attribute '__dict__' of 'A' objects>, '__module__': '__main__', 'b': 2, '__repr__': <function __repr__ at 0x103eb1e60>, 'test': <function test at 0x103eb1758>, '__weakref__': <attribute '__weakref__' of 'A' objects>, '__doc__': None, '__init__': <function __init__ at 0x103eb1140>}
{'c': 3, 'd': 4}
A
```

### 2.4 成员函数
调用成员函数时，其实原理与前一篇分析的函数原理基本一致，只是在类中对PyFunctionObject包装了一层，封装成了PyMethodObject对象，这个对象除了PyFunctionObject对象本身，还新增了class对象和成员函数调用的self参数。PyFunctionObject和一个instance对象通过PyMethodObject对象结合在一起的过程就成为成员函数的绑定。成员函数调用时与一般函数调用机制类似，a.f()函数调用实质就是带了一个位置参数(instance对象a)的一般函数调用。

```
class A(object):
  def f(self):
    pass

a = A()
print A.f # <unbound method A.f>
print a.f # <bound method A.f of <__main__.A object at 0x10d8616d0>>
```

## 3 Python属性选择算法
再谈到属性选择算法之前，需要再说明下descriptor。descriptor分为两种，如下:

- data descriptor: type中定义了__get__和__set__的descriptor。
- no data descriptor: type中只定义了__get__的descriptor。

Python属性选择算法大致规则如下:
- Python虚拟机按照instance属性和class属性顺序选择属性，instance属性优先级高。
- 如果在class属性中发现同名的data descriptor，则data descriptor优先级高于instance属性。

```
#1.data descriptor优先级高于instance属性
class A(list):
  def __get__(self, obj, cls):
    return 'A __get__'

  def __set__(self, obj, value):
    print 'A __set__'
    self.append(value)

class B(object):
  value = A()

b = B()
b.value = 1
print b.value # A.__get__
print b.__class__.__dict__['value'] # [1]
print b.__dict__['value'] # 报错

#2.instance属性优先级高于no data descriptor
class A(list):
  def __get__(self, obj, cls):
    return 'A __get__'

class B(object):
  value = A()

b = B()
b.value = 1
print b.value # 1
print b.__class__.__dict__['value'] # []
print b.__dict__['value'] # 1
```

## 4 其他
Python对象原理还有些不甚明了的地方，暂时记录到这里，后续再补充了。笔记来自《python源码剖析》一书的12章。
