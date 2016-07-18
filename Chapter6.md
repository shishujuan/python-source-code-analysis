#Python源码剖析笔记6-函数机制
> Python的函数机制是很重要的部分，很多时候用python写脚本，就是几个函数简单解决问题，不需要像java那样必须弄个class什么的。

# 1 函数对象PyFunctionObject
PyFunctionObject对象的定义如下：
```
typedef struct {
    PyObject_HEAD
    PyObject *func_code;	/* A code object */
    PyObject *func_globals;	/* A dictionary (other mappings won't do) */
    PyObject *func_defaults;	/* NULL or a tuple */
    PyObject *func_closure;	/* NULL or a tuple of cell objects */
    PyObject *func_doc;		/* The __doc__ attribute, can be anything */
    PyObject *func_name;	/* The __name__ attribute, a string object */
    PyObject *func_dict;	/* The __dict__ attribute, a dict or NULL */
    PyObject *func_weakreflist;	/* List of weak references */
    PyObject *func_module;	/* The __module__ attribute, can be anything */
} PyFunctionObject;
```
先说一下PyFunctionObject中几个重要的变量，func_code, func_globals。其中func_code是函数对象对应的PyCodeObject，而func_globals则是函数的global名字空间，其实这个值是从上一层PyFrameObject传递而来。func_defaults是存储函数默认值的，后面分析函数参数的时候会提到，func_closure与闭包相关，后面也会提到。

```
###func.py
def f():
  print "Function"

f()
```
如上面例子func.py，该文件编译后对应2个```PyCodeObject```对象，一个是func.py本身，一个是函数f。而PyFunctionObject则是在执行字节码```def f():```时通过```MAKE_FUNCTION```指令生成。创建PyFunctionObject对象时，会将函数f对应的PyCodeObject对象和当前PyFrameObject对象传入作为参数，最终也就是赋值给PyFunctionObject中的func_code和func_globals字段了。在调用函数时，会将```PyFunctionObject```对象传入到```fast_function```函数中，最终根据PyFunctionObject对象的func_code和func_globals字段构建新的栈帧对象```PyFrameObject```，然后调用```PyEval_EvalFrameEx```在新的栈帧中执行函数字节码。其中PyEval_EvalFrameEx函数在之前的Python执行原理中有提到过，当时提到的```PyEval_EvalCodeEx```函数其实也是创建了新的栈帧对象PyFrameObject然后执行PyEval_EvalFrameEx函数。

#2 函数调用栈帧
函数调用通过栈帧来建立关联，每个被调用函数的栈帧```PyFrameObject```会通过f_back指针指向调用函数。而local，global以及builtin名字空间，local名字空间针对新的栈帧是全新的，而global名字空间则是由创建PyFrameObject时从PyFunctionObject传递过来。builtin名字空间则是共享调用者栈帧的（如果该栈帧是初始栈帧，则会先获取builtin字典用于设置PyFrameObject的f_builtins字段）。

这里可以回顾一下C语言中的函数调用的栈帧关系。如下面的代码，对应的栈帧结构如图所示。在调用函数时，会先把函数参数会压入当前函数的栈帧中，每个函数都有自己的栈帧，由于esp会变化，所以其他函数会通过ebp来索引函数参数。

![图1 c语言函数栈帧](http://upload-images.jianshu.io/upload_images/286774-12c6c63f2f9ddf54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
//函数调用栈帧测试代码func.c
int bar(int c, int d)
{
	int e = c + d;
	return e;
}

int foo(int a, int b)
{
	return bar(a, b);
}

int main(void)
{
	foo(2, 3);
	return 0;
}
```
那么python中是如何来模拟函数参数传递的呢？从C语言函数调用过程可以知道，函数调用前，函数参数会先压入到调用函数的栈帧中，而被调用函数则根据ebp来取参数。这里先回顾下PyFrameObject对象的结构，函数调用与PyFrameObject有着千丝万缕的联系。
```
typedef struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;	/* previous frame, or NULL */
    PyCodeObject *f_code;	/* code segment */
    PyObject *f_builtins;	/* builtin symbol table (PyDictObject) */
    PyObject *f_globals;	/* global symbol table (PyDictObject) */
    PyObject *f_locals;		/* local symbol table (any mapping) */
    PyObject **f_valuestack;	/* points after the last local */
    PyObject **f_stacktop;
    PyObject *f_trace;		/* Trace function */

    PyObject *f_exc_type, *f_exc_value, *f_exc_traceback;

    PyThreadState *f_tstate;
    int f_lasti;		/* Last instruction if called */
   
    int f_lineno;		/* Current line number */
    int f_iblock;		/* index in f_blockstack */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1];	/* locals+stack, dynamically sized */
} PyFrameObject;
```
PyFrameObject对象中，f_valuestack指向运行时栈的栈底，而f_stacktop则是指向栈顶，在往运行时栈中压入函数参数时，f_stacktop会变化，这两个变量有点类似C里面的ebp和esp。f_localsplus则是指向```局部变量+Cell对象+Free对象+运行时栈```，其内存布局如图2所示。其中cell对象和free对象在闭包中用到，后面再看，这里主要说说局部变量和运行时栈。python在调用函数之前，会先将函数对象，函数参数压入到当前栈帧的运行时栈中，而在执行函数时，会新建一个PyFrameObject栈帧对象，然后将函数参数拷贝到新栈帧的存储局部变量的那块空间中(也就是f_localsplus执行的那块内存)，接着才会调用PyEval_EvalFrameEx执行被调用函数的代码。

![图2 f_localsplus内存布局](http://upload-images.jianshu.io/upload_images/286774-9ad817343843d8c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看下面的func2.py，通过这个例子可以来看一下函数调用流程。例子代码和对应字节码如下。
```
#func2.py
def f(name, age):
    age += 5
    print '%s is %s old' % (name, age)
f('ssj', 18)

##字节码
In [1]: import dis

In [2]: source = open('func2.py').read()

In [3]: co = compile(source, 'func2.py', 'exec')

In [4]: dis.dis(co)
  1           0 LOAD_CONST               0 (<code object f at 0x10776faf8, file "func2.py", line 1>)
              3 MAKE_FUNCTION            0
              6 STORE_NAME               0 (f)

  4           9 LOAD_NAME                0 (f)
             12 LOAD_CONST               1 ('ssj')
             15 LOAD_CONST               2 (18)
             18 CALL_FUNCTION            2
             21 POP_TOP             
             22 LOAD_CONST               3 (None)
             25 RETURN_VALUE 

In [5]: dis.dis(co.co_consts[0])
  2           0 LOAD_FAST                1 (age)
              3 LOAD_CONST               1 (5)
              6 INPLACE_ADD         
              7 STORE_FAST               1 (age)
              ......
```
可以看到```def f(name, age):```跟之前说过的一样，字节码就是通过```MAKE_FUNCTION```指令创建PyFunctionObject对象并存储到local名字空间，对应的符号为函数名f，如果函数有默认参数，在```MAKE_FUNCTION```指令中还会设置默认参数到```func_defaults```字段。而准备调用函数时，则是先讲函数对象和函数参数执行函数时，会将函数参数压栈，然后才通过```CALL_FUNCTION```指令调用函数。在调用PyEval_EvalFrameEx执行函数代码前，创建新的栈帧后，会先将函数参数拷贝到f_localsplus指向的那片局部变量空间中，然后才真正执行函数f调用代码。执行函数f时，会将age参数压入栈然后加上5，然后存储到f_localsplus的第二个字段（第一个字段为name字符串"ssj"）。函数参数位置变化如下图所示。

![图3 函数参数位置变化](http://upload-images.jianshu.io/upload_images/286774-7cc3a13e94db4d27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#3 函数执行时名字空间
还是看第一节中给的例子func.py，其对应的字节码如下，其实定义函数```def f():```就是用函数对应的PyCodeObject和栈帧对应的f_globals构建PyFunctionObject对象，然后通过STORE_NAME指令将PyFunctionObject对象与函数名f关联并存储到local名字空间。函数f对应的PyCodeObject可以通过co.co_consts[0]获取并查看。

```
In [1]: source = open('func.py').read()

In [2]: import dis

In [3]: co = dis.dis(source, 'func.py', 'exec')
 1           0 LOAD_CONST               0 (<code object f at 0x1107688a0, file "func.py", line 1>)
              3 MAKE_FUNCTION            0
              6 STORE_NAME               0 (f)

  4           9 LOAD_NAME                0 (f)
             12 CALL_FUNCTION            0
             15 POP_TOP             
             16 LOAD_CONST               1 (None)
             19 RETURN_VALUE  

In [10]: dis.dis(co.co_consts[0])
  2           0 LOAD_CONST               1 ('Function')
              3 PRINT_ITEM          
              4 PRINT_NEWLINE       
              5 LOAD_CONST               0 (None)
              8 RETURN_VALUE        
```

之前有说过python中分为local，global，builtin名字空间，函数执行时的名字空间略有不同。其global名字空间我们可以看到是通过PyFunctionObject从上一层栈帧传递来的，而local名字空间则是赋值为NULL，也就是函数中并没有用到local名字空间，那么问题来了，函数中的那些局部变量是怎么访问到的呢？那其实在函数中局部变量是通过LOAD_FAST指令(这个指令下一节会分析)来访问的，也就是说它访问的是f_localsplus的内存空间，不需要动态查找f_locals这个PyDictObject，静态的方法可以提供效率。

#4 函数参数
Python中函数参数分为位置参数，键参数以及扩展位置参数和扩展键参数。位置参数就是之前我们例子中的参数，而键参数则是在调用函数指定参数的值。而扩展位置参数和扩展键参数格式则是类似```*lst```和```**kwargs```。位置参数还能设置默认值，如果有默认值，默认值是在```MAKE_FUNCTION```指令赋值给func_defaults的。

下面的例子可以看到这几种参数的用法。扩展位置参数在python内部是通过一个元组对象存储的，不管最终传递了几个参数。而扩展键参数在python内部则是通过一个字典对象存储的。对于像 ```def f(a, b, *lst):```这样的函数，如果调用函数时参数为```f(1,2,3,4)```，其实在PyCodeObject对象中的```co_argcount=2, co_nlocals=3```。co_argcount是位置参数的个数，而co_nlocals是局部变量数目，包括位置参数在内。

```
##params1.py 位置参数和键参数
def f(a, b):
    print a, b
f(1, 2)
f(b=2, a=1)

##params2.py 位置参数，扩展位置参数，扩展键参数
def f(value, *lst, **kwargs):
    print value # -1
    print lst  # (1,2)
    print kwargs # {'a':3, 'b':4}
f(-1, 1, 2, a=3, b=4)

##params3.py 位置参数默认值
def f(lst = []):
    lst.append(3) 
    print lst 
f() #打印[3]
f() #打印[3,3]
```
最后还要提到的一点的是，函数参数默认值是在定义函数时设置的。如例子中的params3.py所示，如果指定了参数默认值，而调用函数时又没有覆盖默认值，则容易出现问题。要解决这个问题，可以在函数f中加个判断```if lst: lst = []```。

#5 闭包和装饰器
之前提到过，PyCodeObject中有两个字段与闭包相关，分别是co_cellvars和co_freevars。其中co_cellvars通常是一个元组，里面保存的是嵌套作用域中使用的变量名集合，而co_freevars也通常是一个元组，里面保存的是外层作用域中的变量名集合。如下面这个闭包的例子，有三个PyCodeObject对象，closure.py本身，函数get_func以及inner_func分布对应一个PyCodeObject。其中get_func的PyCodeObject中的co_cellvars值是元组('value',)，同时，inner_func的PyCodeObject的co_freevars存储的内容也是变量名value。
```
#closure.py 闭包
def get_func():
    value = "inner"
    def inner_func():
         print value
    return inner_func
show_value = get_func()
show_value()
```
可以看下get_func和inner_func的PyFrameObject的内存布局，就可以大致了解闭包的机制了。其实就是在外层函数的局部变量中存储内层嵌套函数inner_func的PyFunctionObject对象，而PyFunctionObject中的func_closure字段是一个存储PyCellObject的元组对象。在执行inner_func时，会先将func_closure中存储的PyCellObject对象拷贝到inner_func的PyFrameObject的free对象中，也就是cell对象后面那块存储空间，这样在inner_func中通过freevars引用到value了（注意，这个freevars不是inner_func的PyCodeObject中的co_freevars，而是PyFrameObject中的对应的内存区域，虽然他们的内容是一致的)。

![图5 闭包机制图示](http://upload-images.jianshu.io/upload_images/286774-9bc30042e2430fe5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

装饰器是基于闭包实现的，可以对一个函数，方法，类进行加工，实现一些额外功能，在实际编码中会经常用到，比如检查用户是否登录，检查输入参数等，就可以用到装饰器来减少冗余代码。下面是一个装饰器的例子：
```
#decorator.py 装饰器
def wrapper(fn):
    def _wrapper():
        print 'wrapper '
        fn()
    return _wrapper

@wrapper
def func():
    print 'real func'

if __name__ == "__main__":
    func()  #输出'wrapper' 'real func'
```
更多装饰器介绍，参见vamei的这篇文章 [Python深入05 装饰器](http://www.cnblogs.com/vamei/archive/2013/02/16/2820212.html)

#6 参考资料
- 《python源码剖析》 主要例子和原理都是参照本书
- [Python快速教程](http://www.cnblogs.com/vamei/archive/2012/09/13/2682778.html)
- 宋劲松 《Linux C语言一站式编程》
- 