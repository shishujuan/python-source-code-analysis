# Python pyc格式分析
> 这篇文章只是纯粹分析python pyc文件格式，主要是关于pyc在文件中的存储方式进行了解析。pyc是python字节码在文件中存储的方式，而在虚拟机运行时环境中对应PyCodeObject对象。关于PyFrameObject以及PyFunctionObject等运行时结构，后续希望学习透彻了能够一并分析。

## 1.示例文件

源文件test.py

```
s = "hello"                                                                                                                                                    

def func():
  a = 3 
  print s

func()
```

通过执行```python pyc_generator.py test``` 可以生成编译好的pyc文件。

```
##pyc_generator.py
import imp                                                                                                                                                  
import sys 


def generate_pyc(name):
  fp, pathname, description = imp.find_module(name)
  try:
    imp.load_module(name, fp, pathname, description)
  finally:
    if fp: 
      fp.close()

if __name__ == "__main__":
  generate_pyc(sys.argv[1])
```

得到test.pyc后，执行```hexdump -C test.pyc```可以得到如下二进制字符流。

```
00000000  03 f3 0d 0a f6 e9 38 55  63 00 00 00 00 00 00 00  |......8Uc.......|
00000010  00 01 00 00 00 40 00 00  00 73 1a 00 00 00 64 00  |.....@...s....d.|
00000020  00 5a 00 00 64 01 00 84  00 00 5a 01 00 65 01 00  |.Z..d.....Z..e..|
00000030  83 00 00 01 64 02 00 53  28 03 00 00 00 74 05 00  |....d..S(....t..|
00000040  00 00 68 65 6c 6c 6f 63  00 00 00 00 01 00 00 00  |..helloc........|
00000050  01 00 00 00 43 00 00 00  73 0f 00 00 00 64 01 00  |....C...s....d..|
00000060  7d 00 00 74 00 00 47 48  64 00 00 53 28 02 00 00  |}..t..GHd..S(...|
00000070  00 4e 69 03 00 00 00 28  01 00 00 00 74 01 00 00  |.Ni....(....t...|
00000080  00 73 28 01 00 00 00 74  01 00 00 00 61 28 00 00  |.s(....t....a(..|
00000090  00 00 28 00 00 00 00 73  1e 00 00 00 2f 55 73 65  |..(....s..../Use|
000000a0  72 73 2f 73 73 6a 2f 50  72 6f 67 2f 70 79 74 68  |rs/ssj/Prog/pyth|
000000b0  6f 6e 2f 74 65 73 74 2e  70 79 74 04 00 00 00 66  |on/test.pyt....f|
000000c0  75 6e 63 03 00 00 00 73  04 00 00 00 00 01 06 01  |unc....s........|
000000d0  4e 28 02 00 00 00 52 01  00 00 00 52 03 00 00 00  |N(....R....R....|
000000e0  28 00 00 00 00 28 00 00  00 00 28 00 00 00 00 73  |(....(....(....s|
000000f0  1e 00 00 00 2f 55 73 65  72 73 2f 73 73 6a 2f 50  |..../Users/ssj/P|
00000100  72 6f 67 2f 70 79 74 68  6f 6e 2f 74 65 73 74 2e  |rog/python/test.|
00000110  70 79 74 08 00 00 00 3c  6d 6f 64 75 6c 65 3e 01  |pyt....<module>.|
00000120  00 00 00 73 04 00 00 00  06 02 09 04              |...s........|
0000012c
```

##2.PyCodeObject结构
PyCodeObject格式如下：

![](http://tech.uc.cn/wp-content/uploads/2013/08/pycfmt.png)

这个图片转自UC技术博客，参见参考资料1。当然这个图片还有些字段没有写出来，比如co_names, co_varnames, co_freevars, co_cellvars,co_filename, co_name, co_firstlineno, co_lnotab。


##3.Pyc格式解析

首先4个字节是magic number，03f30d0a 其中0d0a就是\r\n了接下来4个字节是时间，这里是d2e73855，注意到是小端模式，所以实际是0x5538e7d2,可以发现是我开始编译的时间。然后就是PyCodeObject对象了。首先是对象标识TYPE_CODE，也就是字符c，值为99，即0x63.然后4个字节是全局code block的位置参数个数co_argument，这里是0.再接着4个字节是全局code block中的局部变量个数co_nlocals，这里是0.接着4个字节是code block需要的栈空间co_stacksize，这里值为1.然后4个字节是co_flags，这里是64.

接下来从0x73开始就是code block的字节码序列co_code。注意到它是PyStringObject形式存在，因此，开始写PyStringObject，注意到首先写1个字节的类型标识TYPE_STRING， 即s，对应0x73。然后4个字节标识长度为1a，也就是26个字节。从0x64开始就是co_code内容了。通过dis命令来看一下内容：

```
In [39]: source = open("test.py").read()
In [40]: co = compile(source, 'test.py', 'exec')

In [41]: co.co_consts
Out[41]: ('hello', <code object func at 0x1075710a8, file "test.py", line 3>, None)

In [42]: co.co_names
Out[42]: ('s', 'func')

In [38]: dis.dis(co)
  1           0 LOAD_CONST               0 ('hello') #将co.co_consts[0]即'hello'压栈
              3 STORE_NAME               0 (s) #以co.co_names[0]作为key，将'hello'出栈，然后设置f->f_locas['s'] = 'hello' 

  3           6 LOAD_CONST               1 (<code object func at 0x1075710a8, file "test.py", line 3>) ##将co.co_consts[1]即func的字节码对象压栈
              9 MAKE_FUNCTION            0 ##创建函数对象并压栈
             12 STORE_NAME               1 (func) #f->f_locals['func']=函数对象

  7          15 LOAD_NAME                1 (func) #将f->f_locals['func']即函数对象压栈
             18 CALL_FUNCTION            0 #调用函数，在新栈帧执行
             21 POP_TOP                   ##函数返回值出栈
             22 LOAD_CONST               2 (None) ##None压栈
             25 RETURN_VALUE        ##返回None
```
果然是正好26个字节，其中内容分别对应这些指令，其中第一列是在源码中的行数，第二列是该指令在co_code中的偏移，第三列是opcode，分为有操作数和无操作数两种，是一个字节的整数。第四列是操作数，占两个字节。
那么这些指令对应的就是我们看到的pyc文件中的内容了，具体意义参见代码中的注释。LOAD_CONST指令为0x64，然后两个字节操作数是0.接下来是STORE_NAME指令0x5a，操作数是0.其他以此类推，frame相关内容后面再解析。

接下来从0x28开始是co.co_consts内容，我们知道这是一个PyTupleObject对象，保存着code block的常量，如在前面看到的那样，我们知道它有3个元素，分别是字符串hello，code object对象func以及None。那么PyTupleObject跟PyListObject类似，首先是记录类型标示TYPE_TUPLE，即'(',也就是0x28了。接下来4个字节是长度，这里是3表示有3个元素。然后是元素内容，第一个是'hello‘,它是PyStringObject对象，因此，先写入标记TYPE_INTERNED,即't'，也就是上面的0x74了，然后呢，是写入4个字节的长度，共5个字节，所以这是5，接着就是hello这5个字节。

第二个是code object，好吧，这个就相当于跟之前的流程再来一遍了。
0x63跟之前的一样是TYPE_CODE的标示'c'，然后就是code object的各个字段了。还是来一遍，分别如下

First Header | Second Header 
------------ | -------------  
co_argcount | 0
co_nlocals | 1  
co_stacksize | 1
co_flags | 67
co_code | 标示0x73，即TYPE_STRING。长度0x0f，即15个字节长度。然后从0x64开始就是co_code内容。

下面分析下func的co_code，首先看下dis的结果:

```

In [63]: func.co_nlocals
Out[63]: 1

In [64]: func.co_consts
Out[64]: (None, 3)

In [65]: func.co_names
Out[65]: ('s',)

In [66]: func.co_varnames
Out[66]: ('a',)

In [62]: dis.dis(func)
  4           0 LOAD_CONST               1 (3)  #将func.co_consts[1]即3压栈
              3 STORE_FAST               0 (a)  #存储3在变量a中

  5           6 LOAD_GLOBAL              0 (s) #压入全局变量s
              9 PRINT_ITEM               ##打印s
             10 PRINT_NEWLINE            ##打印换行
             11 LOAD_CONST               0 (None) #None压栈
             14 RETURN_VALUE     ##函数返回None
```

接下来就是func的co_consts字段了，同样是PyTupleObject对象，先是类型标示0x28，然后4个字节为长度2.接着第一个元素是None(N)，即0x4e，然后是第二个元素3，类型标示是TYPE_INT(i)，即0x69.后面4个字节是整数3. 

再接着就是co_names，同样是PyTupleObject对象，显示标示0x28，然后4个字节为长度1，然后字符s是TYPE_INTERNED类型，于是接着是标示't'，即0x74，然后是字符内容s(0x73)。

接下来是co_varnames，同样是PyTupleObject，类型是0x28，然后4个字节为长度1，然后是字符a。

再后面是闭包相关的东西co_freevars，为空的PyTupleObject，类型0x28后面4个字节长度为0.

然后是code block内部嵌套函数引用的局部变量名集合co_cellvars，同样是空的PyTupleObject对象。

接着0x73开始就是co_filename了，这是PyStringObject对象，先是对象标示s，然后是长度30.后面是对应的文件的完整路径"/Users/ssj/Prog/python/test.py"。

接着是co_name，即函数名或者类名，这里就是func了，首先也是对象标示't'(0x74)，后面跟着长度4，然后是’func‘这四个字节。

然后是co_firstlineno，这里直接写的整数3.

然后是字节码指令与源文件行号对应关系co_lnotab，以PyStringObject对象存储。先是标示's'(0x73)，然后是长度4个字节，然后是内容0x00010601.

好吧，至此，func这个code object分析完成。我们回到全局的code object。
全局code object从co_consts[2]开始,这是None，如前面一样，标示为0x4e。接着就是co_names，co_varnames等，分析跟前面func的类似，不再赘述。注意的是这里的co_names对应的's'和’func'类型不再是TYPE_INTERNED，而是TYPE_STRINGREF('R'),值是0x52.还有就是co_lnotab是0x06020904。


##4.参考资料
- [Python程序的执行原理](http://tech.uc.cn/?p=1932)(好文，精简到位，抓住了重点)
- 陈儒《Python源码剖析》(内容很多，有时间值得慢慢研究的好书)