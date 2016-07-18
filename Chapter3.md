#Python源码剖析笔记3-Python执行原理初探

> 之前写了几篇源码剖析笔记，然而慢慢觉得没有从一个宏观的角度理解python执行原理的话，从底向上分析未免太容易让人疑惑，不如先从宏观上对python执行原理有了一个基本了解，再慢慢探究细节，这样也许会好很多。这也是最近这么久没有更新了笔记了，一直在看源码剖析书籍和源码，希望能够从一个宏观层面理清python执行原理。人说读书从薄读厚，再从厚读薄方是理解了真意，希望能够达到这个境地吧，加了个油。

#1 Python运行环境初始化
在看怎么执行之前，先要简单的说明一下python的运行时环境初始化。python中有一个解释器状态对象PyInterpreterState用于模拟进程（后面简称进程对象），另外有一个线程状态对象PyThreadState模拟线程（后面简称线程对象）。python中的PyInterpreterState结构通过一个链表链接起来，用于模拟操作系统多进程。进程对象中有一个指针指向线程集合，线程对象则有一个指针指向其对应的进程对象，这样线程和进程就关联了起来。当然，还少不了一个当前运行线程对象_PyThreadState_Current用来维护当前运行的线程。

##1.1 进程线程初始化
python中调用PyInitialize()函数来完成运行环境初始化。在初始化函数中，会创建进程对象interp以及线程对象并在进程对象和线程对象建立关联，并设置当前运行线程对象为刚创建的线程对象。接下来是类型系统初始化，包括int，str，bool，list等类型初始化，这里留到后面再慢慢分析。然后，就是另外一个大头，那就是系统模块初始化。进程对象interp中有一个modules变量用于维护所有的模块对象,modules变量为字典对象，其中维护(name, module)对应关系，在python中对应着sys.modules。

##1.2 模块初始化
系统模块初始化过程会初始化 ```__builtin__, sys, __main__, site```等模块。在python中，模块对象是以PyModuleObject结构体存在的，除了通用的对象头部，其中就只有一个字典字段md_dict。模块对象中的md_dict字段存储的内容是我们很熟悉的,比如```__name__, __doc__```等属性，以及模块中的方法等。

在```__builtin__```模块初始化中，md_dict中存储的内容就包括内置函数以及系统类型对象，如len,dir,getattr等函数以及int,str,list等类型对象。正因为如此，我们才能在代码中直接用len函数，因为根据LEGB规则，我们能够在```__builtin__```模块中找到len这个符号。几乎同样的过程创建```sys```模块以及```__main__```模块。创建完成后，进程对象```interp->builtins```会被设置为```__builtin__```模块的md_dict字段，即模块对象中的那个字典字段。而```interp->sysdict```则是被设置为sys模块的md_dict字段。

sys模块初始化后，其中包括前面提到过的modules以及path,version,stdin,stdout,maxint等属性，exit,getrefcount,_getframe等函数。注意这里是设置了基本的sys.path（即python安装目录的lib路径等），第三方模块的路径是在site模块初始化的时候添加的。

需要说明的是，```__main__```模块是个特殊的模块，在我们写第一个python程序时，其中的```__name__ == "__main__"```中的```__main__```指的就是这个模块名字。当我们用```python xxx.py```运行python程序时，该源文件就可以当作是名为```__main__```的模块了，而如果是通过其他模块导入，则其名字就是源文件本身的名字，至于为什么，这个在后面运行一个python程序的例子中会详细说明。其中还有一点要说明的是，在创建```__main__```模块的时候，会在模块的字典中插入```("__builtins__", __builtin__ module)```对应关系。在后面可以看到这个模块特别重要，因为在运行时栈帧对象PyFrameObject的f_buitins字段就会被设置为```__builtin__```模块，而栈帧对象的locals和globals字段初始会被设置为```__main__```模块的字典。

另外，```site```模块初始化主要用来初始化python第三方模块搜索路径，我们经常用的sys.path就是这个模块设置的了。它不仅将site-packages路径加到sys.path中，还会把site-packages目录下面的.pth文件中的所有路径加入到sys.path中。

下面是一些验证代码，可以看到sys.modules中果然有了```__builtin__, sys, __main__```等模块。此外，系统的类型对象都已经位于```__builtin__```模块字典中。

```
In [13]: import sys

In [14]: sys.modules['__builtin__'].__dict__['int']
Out[14]: int

In [15]: sys.modules['__builtin__'].__dict__['len']
Out[15]: <function len>

In [16]: sys.modules['__builtin__'].__dict__['__name__']
Out[16]: '__builtin__'

In [17]: sys.modules['__builtin__'].__dict__['__doc__']
Out[17]: "Built-in functions, exceptions, and other objects.\n\nNoteworthy: None is the `nil' object; Ellipsis represents `...' in slices."

In [18]: sys.modules['sys']
Out[18]: <module 'sys' (built-in)>

In [19]: sys.modules['__main__']
Out[19]: <module '__main__' (built-in)>
```

好了，基本工作已经准备妥当，接下来可以运行python程序了。有两种方式，一种是在命令行下面的交互，另外一种是以```python xxx.py```的方式运行。在说明这两种方式前，需要先介绍下python程序运行相关的几个结构。

##1.3 Python运行相关数据结构
python运行相关数据结构主要由PyCodeObject，PyFrameObject以及PyFunctionObject。其中PyCodeObject是python字节码的存储结构，编译后的pyc文件就是以PyCodeObject结构序列化后存储的，运行时加载并反序列化为PyCodeObject对象。PyFrameObject是对栈帧的模拟，当进入到一个新的函数时，都会有PyFrameObject对象用于模拟栈帧操作。PyFunctionObject则是函数对象，一个函数对应一个PyCodeObject,在执行```def test():```语句的时候会创建PyFunctionObject对象。可以这样认为，PyCodeObject是一种静态的结构，python源文件确定，那么编译后的PyCodeObject对象也是不变的；而PyFrameObject和PyFunctionObject是动态结构，其中的内容会在运行时动态变化。

### PyCodeObject对象

python程序文件在执行前需要编译成PyCodeObject对象，每一个CodeBlock都会是一个PyCodeObject对象，在Python中，类，函数，模块都是一个Code Block，也就是说编译后都有一个单独的PyCodeObject对象，因此，一个python文件编译后可能会有多个PyCodeObject对象，比如下面的示例程序编译后就会存在2个PyCodeObject对象，一个对应test.py整个文件，一个对应函数test。关于PyCodeObject对象的解析，可以参见我之前的文章[Python pyc格式解析](http://www.jianshu.com/p/03d81eb9ac9b)，这里就不赘述了。


```
#示例代码test.py
def test():
    print "hello world"

if __name__ == "__main__":
    test()
```

### PyFrameObject对象
python程序的字节码指令以及一些静态信息比如常量等都存储在PyCodeObject中，运行时显然不可能只是操作PyCodeObject对象，因为有很多内容是运行时动态改变的，比如下面这个代码test2.py,虽然1和2处的字节码指令相同，但是它们执行结果显然是不同的，这些信息显然不能在PyCodeObject中存储，这些信息其实需要通过PyFrameObject也就是栈帧对象来获取。PyFrameObject对象中有locals，globals，builtins三个字段对应local，global，builtin三个名字空间，即我们常说的LGB规则，当然加上闭包，就是LEGB规则。一个模块对应的文件定义一个global作用域，一个函数定义一个local作用域，python自身定义了一个顶级作用域builtin作用域，这三个作用域分别对应PyFrameObject对象的三个字段，这样就可以找到对应的名字引用。比如test2.py中的1处的i引用的是函数test的局部变量i，对应内容是字符串“hello world”，而2处的i引用的是模块的local作用域的名字i，对应内容是整数123(注意模块的local作用域和global作用域是一样的)。**需要注意的是，函数中局部变量的访问并不需要去访问locals名字空间，因为函数的局部变量总是不变的，在编译时就能确定局部变量使用的内存位置。**

```
#示例代码test2.py
i = 123                                                                                                                                                       

def test():
  i = 'hello world'
  print i #1

test()
print i #2
```

### PyFunctionObject对象
PyFunctionObject是函数对象，在创建函数的指令MAKE_FUNCTION中构建。PyFunctionObject中有个func_code字段指向该函数对应的PyCodeObject对象，另外还有func_globals指向global名字空间，注意到这里并没有使用local名字空间。调用函数时，会创建新的栈帧对象PyFrameObject来执行函数，函数调用关系通过栈帧对象PyFrameObject中的f_back字段进行关联。最终执行函数调用时，PyFunctionObject对象的影响已经消失，真正起作用的是PyFunctionObject的PyCodeObject对象和global名字空间，因为在创建函数栈帧时会将这两个参数传给PyFrameObject对象。

##1.4 Python程序运行过程浅析
说完几个基本对象，现在回到之前的话题，开始准备执行python程序。两种方式交互式和直接```python xxx.py```虽然有所不同，但最终归于一处，就是启动虚拟机执行python字节码。这里以```python xxx.py```方式为例，在运行python程序之前，需要对源文件编译成字节码，创建PyCodeObject对象。这个是通过PyAST_Compile函数实现的，至于具体编译流程，这就要参看《编译原理》那本龙书了，这里暂时当做黑盒好了，因为单就编译这部分而言，一时半会也说不清楚（好吧，其实是我也没有学好编译原理）。编译后得到PyCodeObject对象，然后调用```PyEval_EvalCode(co, globals, locals)```函数创建PyFrameObject对象并执行字节码了。注意到参数里面的co是PyCodeObject对象，而由于运行PyEval_EvalCode时创建的栈帧对象是Python创建的第一个PyFrameObject对象，所以f_back为NULL，而且它的globals和locals就是```__main__```模块的字典对象。如果我们不是直接运行，而是导入一个模块的话，则还会将python源码编译后得到的PyCodeObject对象保存到pyc文件中，下次加载模块时如果这个模块没有改动过就可以直接从pyc文件中读取内容而不需要再次编译了。

执行字节码的过程就是模拟CPU执行指令的过程一样，先指向PyFrameObject的f_code字段对应的PyCodeObject对象的co_code字段，这就是字节码存储的位置，然后取出第一条指令，接着第二条指令...依次执行完所有的指令。python中指令长度为1个字节或者3个字节，其中无参数的指令长度是1个字节，有参数的指令长度是3个字节（指令1字节+参数2字节）。

python虚拟机的进程，线程，栈帧对象等关系如下图所示：

![py.png](http://upload-images.jianshu.io/upload_images/286774-d8cce06585f5aec7.png)

#2 Python程序运行实例说明
程序猿学习一门新的语言往往都是从hello world开始的，一来就跟世界打个招呼，因为接下来就要去面对程序语言未知的世界了。我学习python也是从这里开始的，只是以前并不去深究它的执行原理，这回是逃不过去了。看看下面的栗子。

```
#示例代码test3.py
i = 1
s = 'hello world'

def test():
    k = 5
    print k
    print s

if __name__ == "__main__":
    test()
```

这个例子代码不多，不过也涉及到python运行原理的方方面面(除了类机制那一块外，类机制那一块还没有理清楚，先不理会)。那么按照之前部分说的，执行```python test3.py```的时候，会先初始化python进程和线程，然后初始化系统模块以及类型系统等，然后运行python程序test3.py。每次运行python程序都是开启一个python虚拟机，由于是直接运行，需要先编译为字节码格式，得到PyCodeObject对象，然后从字节码对象的第一条指令开始执行。因为是直接运行，所以PyCodeObject也就没有序列化到pyc文件保存了。下面可以看下test3.py的PyCodeObject，使用python的dis模块可以看到字节码指令。

```
In [1]: source = open('test3.py').read()

In [2]: co = compile(source, 'test3.py', 'exec')

In [3]: co.co_consts
Out[3]: 
(1,
 'hello world',
 <code object test at 0x1108eaaf8, file "run.py", line 4>,
 '__main__',
 None)

In [4]: co.co_names
Out[4]: ('i', 's', 'test', '__name__')

In [5]: dis.dis(co) ##模块本身的字节码，下面说的整数，字符串等都是指python中的对象，对应PyIntObject，PyStringObject等。
  1           0 LOAD_CONST               0 (1) # 加载常量表中的第0个常量也就是整数1到栈中。
              3 STORE_NAME               0 (i) # 获取变量名i，出栈刚刚加载的整数1，然后存储变量名和整数1到f->f_locals中，这个字段对应着查找名字时的local名字空间。

  2           6 LOAD_CONST               1 ('hello world') 

              9 STORE_NAME               1 (s)  #同理，获取变量名s，出栈刚刚加载的字符串hello world，并存储变量名和字符串hello world的对应关系到local名字空间。

  4          12 LOAD_CONST               2 (<code object test at 0xb744bd10, file "test3.py", line 4>)
             15 MAKE_FUNCTION            0   #出栈刚刚入栈的函数test的PyCodeObject对象，以code object和PyFrameObject的f_globals为参数创建函数对象PyFunctionObject并入栈
             18 STORE_NAME               2 (test)  #获取变量test，并出栈刚入栈的PyFunctionObject对象，并存储到local名字空间。

  9          21 LOAD_NAME                3 (__name__) ##LOAD_NAME会先依次搜索local，global，builtin名字空间，当然我们这里是在local名字空间能找到__name__。
             24 LOAD_CONST               3 ('__main__')
             27 COMPARE_OP               2 (==)  ##比较指令
             30 JUMP_IF_FALSE           11 (to 44) ##如果不相等则直接跳转到44对应的指令处，也就是下面的POP_TOP。因为在COMPARE_OP指令中，会设置栈顶为比较的结果，所以需要出栈这个比较结果。当然我们这里是相等，所以接着往下执行33处的指令，也是POP_TOP。
             33 POP_TOP             

 10          34 LOAD_NAME                2 (test) ##加载函数对象
             37 CALL_FUNCTION            0  ##调用函数
             40 POP_TOP                     ##出栈函数返回值
             41 JUMP_FORWARD             1 (to 45) ##前进1步，注意是下一条指令地址+1，也就是44+1=45
        >>   44 POP_TOP             
        >>   45 LOAD_CONST               4 (None) 
             48 RETURN_VALUE     #返回None


In [6]: dis.dis(co.co_consts[2])  ##查看函数test的字节码
  5           0 LOAD_CONST               1 (5)
              3 STORE_FAST               0 (k) #STORE_FAST与STORE_NAME不同，它是存储到PyFrameObject的f_localsplus中，不是local名字空间。

  6           6 LOAD_FAST                0 (k) #相对应的，LOAD_FAST是从f_localsplus取值
              9 PRINT_ITEM          
             10 PRINT_NEWLINE         #打印输出 

  7          11 LOAD_GLOBAL              0 (s) #因为函数没有使用local名字空间，所以，这里不是LOAD_NAME,而是LOAD_GLOBAL，不要被名字迷惑，它实际上会依次搜索global，builtin名字空间。
             14 PRINT_ITEM          
             15 PRINT_NEWLINE       
             16 LOAD_CONST               0 (None)
             19 RETURN_VALUE        

```

按照我们前面的分析，test3.py这个文件编译后其实对应2个PyCodeObject，一个是本身test3.py这个模块整体的PyCodeObject，另外一个则是函数test对应的PyCodeObject。根据PyCodeObject的结构，我们可以知道test3.py字节码中常量co_consts有5个，分别是整数1，字符串‘hello world'，函数test对应的PyCodeObject对象，字符串```__main__```，以及模块返回值None对象。恩，从这里可以发现，其实模块也是有返回值的。我们同样可以用dis模块查看函数test的字节码。

关于字节码指令，代码中做了解析。需要注意到函数中局部变量如k的取值用的是LOAD_FAST，即直接从PyFrameObject的f_localsplus字段取，而不是LOAD_NAME那样依次从local，global以及builtin查找，这是函数的特性决定的。函数的运行时栈也是位于f_localsplus对应的那片内存中，只是前面一部分用于存储函数参数和局部变量，而后面那部分才是运行时栈使用，这样逻辑上运行时栈和函数参数以及局部变量是分离的，虽然物理上它们是连在一起的。需要注意的是，python中使用了预测指令机制，比如COMPARE_OP经常跟JUMP_IF_FALSE或JUMP_IF_TRUE成对出现，所以如果COMPARE_OP的下一条指令正好是JUNP_IF_FALSE，则可以直接跳转到对应代码处执行，提高一定效率。

此外，还要知道在运行test3.py的时候，模块的test3.py栈帧对象中的f_locals和f_globals的值是一样的，都是```__main__```模块的字典。在test3.py的代码后面加上如下代码可以验证这个猜想。

```
 ... #test3.py的代码
 
if __name__ == "__main__":
    test()
 	print locals() == sys.modules['__main__'].__dict__ # True
 	print globals() == sys.modules['__main__'].__dict__ # True
 	print globals() == locals() # True
```
正式因为如此，所以python中函数定义顺序是无关的，不需要跟C语言那样在调用函数前先声明函数。比如下面test4.py是完全正常的代码，函数定义顺序不影响函数调用，因为在执行def语句的时候，会执行MAKE_FUNCTION指令将函数对象加入到local名字空间，而local和global此时对应的是同一个字典，所以也相当于加入了global名字空间，从而在运行函数g的时候是可以找到函数f的。另外也可以注意到，函数声明和实现其实是分离的，声明的字节码指令在模块的PyCodeObject中执行，而实现的字节码指令则是在函数自己的PyCodeObject中。

```
#test4.py
def g():                                                                                                                                                     
  print 'function g'
  f() 

def f():
  print 'function f'

g()
~      
```