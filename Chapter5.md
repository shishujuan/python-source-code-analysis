# Python源码剖析笔记5-模块机制

> python中经常用到模块，比如```import xxx,from xxx import yyy```这样子，里面的机制也是需要好好探究一下的，这次主要从黑盒角度来探测模块机制，源码分析点到为止，详尽的源码分析见陈儒大神的《python源码剖析》第14章。

# 1 如何导入模块
首先来看一个导入模块的例子。创建一个文件夹demo5，文件夹中有如下几个文件。

```
ssj@ssj-mbp ~/demo5 $ ls
__init__.py math.py     sys.py      test.py
```
根据python规则，因为文件夹下面有__init__.py文件，因此demo5是一个包。各个文件内容如下：

```
#__init__.py
import sys
import math

#math.py
print 'my math'

#sys.py
print 'my sys'

#test.py
import sys
import math
```
好了，问题来了，当我在demo5目录运行```python test.py```的时候，会打印什么结果呢？sys模块和math模块会调用demo5目录下面的还是系统本身的模块呢？结果是只打印出了```my math```，也就是说，sys模块并不是导入的demo5目录下面的sys模块。但是，如果我们不是直接运行test.py，而是导入整个包呢？结果大为不同，当我们在demo5上层目录执行```import demo5```时，可以发现打印出了```my sys```和```my math```，也就是说，导入的都是demo5目录下面的两个模块。出现这两个不同结果就是python模块和包导入机制导致的。下面来分析下python模块和包导入机制。

# 2 Python模块和包导入原理
python模块和包导入函数调用路径是```builtin___import__->import_module_level->load_next->import_submodule->find_module->load_module```，本文不打算分析所有的函数，只摘出几处关键代码分析。

```builtin___import__```函数解析import参数，比如```import xxx```和```from yyy import xxx```解析后获取的参数是不一样的。然后通过```import_module_level```函数解析模块和包的树状结构，并调用```load_next```来导入模块。而```load_next```调用```import_submodule```来查找并导入模块。注意到如果是从包里面导入模块的话，```load_next```会**先用包含包名的完整模块名**调用```import_submodule```来寻找并导入模块，如果找不到，则只用模块名来寻找并导入模块。```import_submodule```会先根据**模块完整名fullname**来判断是否是系统模块，即之前说过的sys.modules是否有该模块，比如sys，os等模块，如果是系统模块，则直接返回对应模块。否则根据模块路径调用```find_module```搜索模块并调用```load_module```函数导入模块。注意到如果不是从包中导入模块，find_module中会判断模块是否是内置模块或者扩展模块(注意到这里的内置模块和扩展模块是指不常用的系统模块，比如imp和math模块等)，如果是则直接初始化该内置模块并加入到之前的备份模块集合extensions中。否则需要先后搜索模块包的路径和系统默认路径是否有该模块，如果都没有搜索到该模块，则报错。找到了模块，则初始化模块并将模块引用加入到```sys.modules```中。

```load_module```这个函数需要额外说明下，该函数会根据模块类型不同来使用不同的加载方式，基本类型有PY_SOURCE, PY_COMPILED,C_BUILTIN, C_EXTENSION,PKG_DIRECTORY等。PY_SOURCE指的就是普通的py文件，而PY_COMPILED则是指编译后的pyc文件，如果py文件和pyc文件都存在，则这里的类型为PY_SOURCE,你可能会有点纳闷了，这样岂不是影响效率了么？其实不然，这是为了保证导入的是最新的模块代码，因为在load_source_module中会判断pyc文件是否过时，如果没有过时，还是会在这里导入pyc文件的，所以性能上并不会有太多影响。而C_BUILTIN指的是系统内置模块，比如imp模块，C_EXTENSION指的是扩展模块，通常是以动态链接库形式存在的，比如math.so模块。PKG_DIRECTORY则是指导入的是包，比如导入demo5包，会先导入包demo5本身，然后导入__init__.py模块。
```
/*load_next函数部分代码*/
static PyObject *load_next() {
    .......
    result = import_submodule(mod, p, buf); //p是模块名,buf是包含包名的完整模块名
    if (result == Py_None && altmod != mod) {
	    result = import_submodule(altmod, p, p);
    }
    .......
}
```
```
/*import_submodule部分代码*/
static PyObject *
import_submodule(PyObject *mod, char *subname, char *fullname)
{
	PyObject *modules = PyImport_GetModuleDict();
	PyObject *m = NULL;

	
	if ((m = PyDict_GetItemString(modules, fullname)) != NULL) {
		Py_INCREF(m);
	}
	else {
                ......
		if (mod == Py_None)
			path = NULL;
		else {
			path = PyObject_GetAttrString(mod, "__path__");
                        ......
		}
		.......
		fdp = find_module(fullname, subname, path, buf, MAXPATHLEN+1,
				  &fp, &loader);
		.......
		m = load_module(fullname, fp, buf, fdp->type, loader);
                .......
		if (!add_submodule(mod, m, fullname, subname, modules)) {
			Py_XDECREF(m);
			m = NULL;
		}
	}
	return m;
}

```
接下来就需要解释下第一节中提出的问题了，首先直接```python test.py```的时候，那么先后导入```sys模块和math模块```，由于是直接导入模块，则全名就是sys，在导入sys模块的时候，虽然当前目录下有sys模块，但是sys模块是系统模块，所以会在import_submodule中直接返回系统的sys模块。而math模块不是系统预先加载的模块，所以会在当前目录下找到并加载。

而如果使用了包机制，我们```import demo5```时，则此时会先加载demo5包本身，然后加载```__init__.py```模块，__init__.py中会加载sys和math模块，由于是通过包来加载，所以fullname会变成demo5.sys和demo5.math。显然在判断的时候，demo5.sys不在系统预先加载的模块sys.modules中，因此最终会加载当前目录下面的sys模块。math则跟前面情况类似。

# 3 模块和名字空间
在导入模块的时候，会在名字空间中引入对应的名字。注意到导入模块和设置的名字空间的名字时不一样的，需要注意区分下。下面给个栗子，这里有个包foobar，里面有```a.py, b.py,__init__.py```。
``` 
In [1]: import sys

In [2]: sys.modules['foobar']
---------------------------------------------------------------------------
KeyError                                  Traceback (most recent call last)
<ipython-input-2-9001cd5d540a> in <module>()
----> 1 sys.modules['foobar']

KeyError: 'foobar'

In [3]: import foobar
import package foobar

In [4]: sys.modules['foobar']
Out[4]: <module 'foobar' from 'foobar/__init__.pyc'>

In [5]: import foobar.a
import module a

In [6]: sys.modules['foobar.a']
Out[6]: <module 'foobar.a' from 'foobar/a.pyc'>

In [7]: locals()['foobar']
Out[7]: <module 'foobar' from 'foobar/__init__.pyc'>

In [8]: locals()['foobar.a']
---------------------------------------------------------------------------
KeyError                                  Traceback (most recent call last)
<ipython-input-8-059690e6961a> in <module>()
----> 1 locals()['foobar.a']

KeyError: 'foobar.a'

In [9]: from foobar import b
import module b

In [10]: locals()['b']
Out[10]: <module 'foobar.b' from 'foobar/b.pyc'>

In [11]: sys.modules['foobar.b']
Out[11]: <module 'foobar.b' from 'foobar/b.pyc'>

In [12]: sys.modules['b']
---------------------------------------------------------------------------
KeyError                                  Traceback (most recent call last)
<ipython-input-13-1df8d2911c99> in <module>()
----> 1 sys.modules['b']

KeyError: 'b'
```
我们知道，导入的模块都会加入到```sys.modules```字典中。当我们导入模块的时候，可以简单分为以下几种情况，具体原理可以参见源码：
- import foobar.a
这是直接导入模块a，那么在sys.modules中存在foobar和foobar.a，但是在local名字空间中只存在foobar，并没有foobar.a。这是由import机制决定的，在导入模块的代码中可以看到针对foobar.a最终存储到名字空间的只有foobar。
- from foobar import b
这种情况存储到sys.modules的也只有foobar(前面已经导入不会重复导入了)和foobar.b。local名字空间只有b，没有foobar，也没有foobar.b。
- import foobar.a as A
这种情况sys.modules中还是foobar和foobar.a，而local名字空间只有A，没有foobar，更没有foobar.a。如果我们执行```del A```删除符号A，则名字空间不在有符号A，但是在sys.modules中还是存在foobar和foobar.a的。

# 4 其他
需要提到的一点是，如果我们修改了某个py文件，然后reload该模块，则删除的符号并不会更新，而只是会加入新增加的符号或者更新已经有的符号。比如下面这个例子，我们加入 b = 2后reload模块reloadtest，可以看到模块中多了符号b，而我们删除b = 2加入c=3后，发现符号b还是在模块reloadtest中，并没有删除。这是python内部reload机制决定的，在reload操作时，python内部实现是找到原模块的字典，并更新或添加符号，并不删除原有的符号。
```
#初始代码 a = 1
In [1]: import reloadtest 

In [2]: import sys

In [3]: dir(sys.modules['reloadtest'])
Out[3]: ['__builtins__', '__doc__', '__file__', '__name__', '__package__', 'a']

##新增一行代码 b = 2 
In [4]: reload(reloadtest)
Out[4]: <module 'reloadtest' from 'reloadtest.py'>

In [5]: dir(sys.modules['reloadtest'])
Out[5]: ['__builtins__', '__doc__', '__file__', '__name__', '__package__', 'a', 'b']

##删除代码 b = 2，新增 c = 3
In [6]: reload(reloadtest)
Out[6]: <module 'reloadtest' from 'reloadtest.py'>

In [7]: dir(sys.modules['reloadtest'])
Out[7]: 
['__builtins__',
 '__doc__',
 '__file__',
 '__name__',
 '__package__',
 'a',
 'b',
 'c']
```
