# Python源码剖析笔记4-内建数据类型
> Python内建数据类型包括整数对象PyIntObject，字符串对象PyStringObject，列表对象PyListObject以及字典对象PyDictObject等。整数对象之前已经分析过了，这一篇文章准备分析下余下几个对象，这次在《python源码剖析》中已经写的很详细的部分就不赘述了，主要是总结一些之前看书时疑惑的地方。

## 1 整数对象-PyIntObject
参见 [python整数对象](http://www.jianshu.com/p/0136ed90cd46)。


## 2 字符串对象-PyStringObject
### 2.1 基本定义
python中的字符串对象PyStringObject，对应的类型对象是PyString_Type。PyStringObject对象的定义如下：

```
#define PyObject_VAR_HEAD   \                                                                                                                                 
  Py_ssize_t ob_refcnt;   \
  struct _typeobject *ob_type;  \
  Py_ssize_t ob_size;
  
 
typedef struct {
    PyObject_VAR_HEAD //对象头
    long ob_shash; //字符串哈希值
    int ob_sstate; //对象状态
    char ob_sval[1]; //字符串内容

    /* Invariants:
     *     ob_sval contains space for 'ob_size+1' elements.
     *     ob_sval[ob_size] == 0.
     *     ob_shash is the hash of the string or -1 if not computed yet.
     *     ob_sstate != 0 iff the string object is in stringobject.c's
     *       'interned' dictionary; in this case the two references
     *       from 'interned' to this object are *not counted* in ob_refcnt.
     */
} PyStringObject;
```
字符串长度在头部PyObject_VAR_HEAD的ob_size字段中维护，而ob_sval则是指向一段长度为ob_size+1个字节的内存，比如字符串'hello'，ob_size=5，而ob_sval长度为6，ob_sval[6] = '\0'。 ob_sstate是字符串状态，标示字符串是否经过intern机制处理。ob_shash是字符串的哈希值，在字典以及字符串比较等多处有用到这个哈希值。

### 2.2 字符串interned机制

当然在字符串对象中一个比较重要的就是intern机制。那么问题来了，什么样的字符串才会interned呢？实验一下先，可以发现如果字符串有空格是不会被interned的，实际上，字符串中的字符必须都是属于```"0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ_abcdefghijklmnopqrstuvwxyz"```才会interned。例子中的```hello world```因为有空格不在interned字符集中，所以该字符串不会interned。这个是在构建PyCodeObject对象的时候进行的。之前已经分析过，python的py文件需要编译成字节码执行，当然直接执行和import导入模块有所不同，不过都会构建PyCodeObject对象。在构建PyCodeObject对象函数PyCode_New(Objects/codeObject.c文件)中，会执行变量名、常量等字符串的interned操作。

另外一个需要说明的是，在编译py文件成字节码并保存到pyc文件过程中，字符串对象分为三种情况处理。其一是非interned字符串，比如刚刚说的```hello world```字符串，对象标识是s。其二是interned字符串，比如```hello```，对象标识是t。其三是已经interned过的字符串，在pyc中记录的对象标识是R。之所以有R标记的字符串，是为了节省空间，因为它最终只记录一个字符串的偏移位置。比如之前已经有了字符串'hello',则写入标记s以及字符串内容'hello',第二次遇到'hello'时则只是写入标记R以及'hello'在常量元组co_consts中的索引值。这样从pyc内容中构建PyCodeObject对象的时候，根据R标识类型字符串记录的索引得到字符串。

```
#测试intern机制test_interned.py
a = 'hello world'
b = 'hello'

def test():
    c = 'hello world'
    d = 'hello'
    print c is a  #False，'hello world' 没有interned
    print d is b  #True， 'hello' 已经interned
    
test()
```

关于R标记，还要多说一点，我之前看书的时候也有点疑惑什么情况下会用到R标记呢，因为你在一个模块里面对一个字符串多次引用，在PyCodeObject对象的co_consts中还是只会存在一份的。比如下面的代码，显然s和t针对test.py对应的PyCodeObject对象，只会有一个常量'hello',虽然这个字符串会被interned，那么R标记是用在哪里呢？其实是用在另外的PyCodeObject中。比如下面的代码，对应两个PyCodeObject，其中test_stringref.py本身一个，以及函数test对应一个PyCodeObject。编译后得到的pyc文件内容如下所示，根据前面的文章pyc格式分析，可以看到在test_stringref.py本身对应的PyCodeObject中，co_consts为```('hello', <code object test at 0x10c130af8, file "str.py"", line 4>, None) ```，这里看到s和t引用的是同一个字符串，这一点通过字节码指令也可以看到。 在pyc中'hello'存储的是标识t以及字符串内容。而函数test对应的PyCodeObject中，对应的co_consts为```(None, hello)```，但是在pyc中对应字符串hello存储的是标记R以及索引0。

此外，如果直接运行```python xxx.py```，虽然也会编译成PyCodeObject对象，但是不会生成pyc文件，也不会有R标识类型这些东西了，不过interned机制在运行的时候同样会生效。不同的地方在于，如果是从pyc文件运行会根据R标识类型来将对应字符串指向同一个对象，而如果是直接运行的，则需要通过interned字典来对后面遇到的相同的可以interned字符串对象赋值为interned字符串对象的地址，进行并回收后面遇到的那个字符串。

单个字符和空字符都会interned，这个可以很简单的验证。

```
#测试字符串类型标识 test_stringref.py
s = 'hello'
t = 'hello'
def test():
    k = 'hello'
    
#test_stringref.py对应的字节码
          0 LOAD_CONST          0 (0)
          3 STORE_NAME          0 (0)
          6 LOAD_CONST          0 (0)
          9 STORE_NAME          1 (1)
         12 LOAD_CONST          1 (1)
         15 MAKE_FUNCTION       0
         18 STORE_NAME          2 (2)
         21 LOAD_CONST          2 (2)
         24 RETURN_VALUE   
   
#test_stringref.pyc文件
00000000  03 f3 0d 0a 78 3e 99 55  63 00 00 00 00 00 00 00  |....x>.Uc.......|
00000010  00 01 00 00 00 40 00 00  00 73 19 00 00 00 64 00  |.....@...s....d.|
00000020  00 5a 00 00 64 00 00 5a  01 00 64 01 00 84 00 00  |.Z..d..Z..d.....|
00000030  5a 02 00 64 02 00 53 28  03 00 00 00 74 05 00 00  |Z..d..S(....t...|
00000040  00 68 65 6c 6c 6f 63 00  00 00 00 01 00 00 00 01  |.helloc.........|
00000050  00 00 00 43 00 00 00 73  0a 00 00 00 64 01 00 7d  |...C...s....d..}|
00000060  00 00 64 00 00 53 28 02  00 00 00 4e 52 00 00 00  |..d..S(....NR...|
00000070  00 28 00 00 00 00 28 01  00 00 00 74 01 00 00 00  |.(....(....t....|
00000080  6b 28 00 00 00 00 28 00  00 00 00 73 1d 00 00 00  |k(....(....s....|
00000090  2f 55 73 65 72 73 2f 73  73 6a 2f 50 72 6f 67 2f  |/Users/ssj/Prog/|
000000a0  70 79 74 68 6f 6e 2f 73  74 72 2e 70 79 74 04 00  |python/str.pyt..|
000000b0  00 00 74 65 73 74 04 00  00 00 73 02 00 00 00 00  |..test....s.....|
000000c0  01 4e 28 03 00 00 00 74  01 00 00 00 73 74 01 00  |.N(....t....st..|
000000d0  00 00 74 52 02 00 00 00  28 00 00 00 00 28 00 00  |..tR....(....(..|
000000e0  00 00 28 00 00 00 00 73  1d 00 00 00 2f 55 73 65  |..(....s..../Use|
000000f0  72 73 2f 73 73 6a 2f 50  72 6f 67 2f 70 79 74 68  |rs/ssj/Prog/pyth|
00000100  6f 6e 2f 73 74 72 2e 70  79 74 08 00 00 00 3c 6d  |on/str.pyt....<m|
00000110  6f 64 75 6c 65 3e 01 00  00 00 73 04 00 00 00 06  |odule>....s.....|
00000120  01 06 02                                          |...|
00000123
```

### 2.3 字符串拼接效率问题
另外一个需要注意的就是字符串拼接的效率问题。如果是简单的 ```s1+s2+s3```这样拼接，那么每次拼接都要分配一次内存，这样需要分配两次内存。而如果通过```''.join([s1, s2, s3])```来拼接，则只需要分配一次内存，在拼接字符串较多的时候，通过join操作拼接字符串效率会有大幅提高。


## 3 列表对象-PyListObject
Python中的List对象实现有点类似STL中的vector，依托的是数组形式来实现列表。定义如下：

```
typedef struct {
    PyObject_VAR_HEAD
    /* Vector of pointers to list elements.  list[0] is ob_item[0], etc. */
    PyObject **ob_item;
    Py_ssize_t allocated;
} PyListObject;
```
可以看到前面跟PyStringObject是一样的头部，其中的ob_size是当前列表元素数目，而allocated是分配的空间大小， ```ob_size <= allocated```，也就是说一般情况下会多分配一点空间，以减少多次分配带来性能问题。列表初始化分为两部分，列表本身结构初始化和列表维护的对象列表ob_item初始化。当然，列表本身初始化也采用了缓冲池机制，如果缓冲池列表中有空闲的列表可以用，就可以直接拿来用而不需要再次分配内存了。**当然，虽然列表结构本身可以通过缓冲池复用，但是其中维护的对象列表ob_item是不会复用的，从缓冲池中得到列表结构后还需要给ob_item分配空间（空间大小为 ```size * sizeof(PyObject *),size为创建的列表大小```）**。

列表分配的空间大小allocated在通过insert，append操作插入元素时或者通过remove, del操作时会进行调整，也就是说即可能变大也可能变小。调整列表大小的函数是```list_resize()```，调整条件如下:

```
(1) newsize <= allocated && newsize >= allocated / 2:简单调整ob_size的值，不需要扩容。
(2) 否则，调整大小为 new_allocated = (newsize >> 3) + (newsize < 9 ? 3: 6) + newsize.
```
那么，假定有一个语句如下```list = [1]```，则字节码其实就是创建一个大小为1(ob_size和allocated此时都是1)的列表，然后将列表list中的ob_item的第0个元素设置为整数对象1。**需要注意的是，当你创建一个空列表然后直接赋值则会出错的，比如下面那样，这是因为你创建一个空列表时ob_item被设置为NULL，并没有分配内存，因此会报错。**

```
##test1.py 未分配对象列表内存导致赋值错误
list = []
list[0] = 1 #错误
```
还是继续刚刚那个栗子，现在有列表list=[1]，然后执行list.append(2)，此时ob_size=2，而根据上面的策略，allocated调整为5.再执行list.append(3)插入3，则此时ob_size=3，而allocated=5不变。如果接着list.remove(2)，则ob_size=2，allocated=5不变。此时如果再执行list.remove(3)，则根据上面的调整公式，ob_size=1，allocated减小到4。


## 4 字典对象-PyDictObject
python字典对象采用的散列冲突解决方法为开放定址法，不同于STL中的开链法。python采用的开放定址法在发生散列冲突时，会根据一个冲突探测函数计算下一个探测的位置，直到找到一个不冲突的位置。而在删除元素的时候，并不会直接删除，而是设置一个dummy标记，这样可以保证在冲突探测的时候不出错，此外这个dummy标记的元素下次插入新元素时可以被再次利用。

字典中的一个键值对称为一个entry，字典由PyDictEntry的数组构成。定义如下：

```
typedef struct {
  Py_ssize_t me_hash; //me_key的哈希值
  PyObject *me_key; 
  PyObject *me_value;
} PyDictEntry;  

#define PyDict_MINSIZE 8 
typedef struct _dictobject PyDictObject;
struct _dictobject {
  PyObject_HEAD
  Py_ssize_t ma_fill;  /* # Active + # Dummy */
  Py_ssize_t ma_used;  /* # Active */
  Py_ssize_t ma_mask;
  PyDictEntry *ma_table;
  PyDictEntry *(*ma_lookup)(PyDictObject *mp, PyObject *key, long hash);
  PyDictEntry ma_smalltable[PyDict_MINSIZE];
};
```

键值对有active， dummy以及unused三个状态。初始为unused状态，```me_key == NULL, me_value == NULL```。而如果是active状态，则```me_key != NULL, me_key != dummy, me_value != NULL```,如果是dummy状态，则```me_key == dummy, me_value == NULL```.三个状态的转换则是：执行插入操作则由unused或者dummy状态变成active状态，执行删除操作则从active状态变成dummy状态。

PyDictObject定义如上面所示，其中ma_fill是元素总数目，包括active和dummy状态的键值对。而ma_used是active键值对数目，ma_mask用来定位元素在数组中的索引，大小为数组长度减一。如果数组ma_table为8，则ma_mask值为7.注意，这里还有个ma_smalltable，大小初始为8,初始化时，字典的ma_table会指向ma_smalltable，当装载率大于等于2/3时，字典扩容。```(即ma_fill / (ma_mask + 1)) >=2/3)```，会将ma_table指向新分配的空间，并搬移键值对到新的table中。

PyDictObject也采用了缓冲池机制，其原理类似PyListObject，这里不再赘述。需要额外提到的一点是字典中的元素搜索机制，这里说下常用的lookdict_string函数，其针对的是字符串键的查找。查找流程如下：

```
a. 查找的时候会先根据字符串hash值获取键值对索引，如果对应的键值对entry为unused状态，则表示搜索失败，如果freeslot不为NULL，则返回freeslot（freeslot指向dummy状态的entry），否则返回entry。
b. 如果entry为其他状态，则检查me_key引用是否相同，相同则返回该entry.
c. 如果me_key引用不同，则检查me_key值是否相同(即比较hash值以及me_key的值)，如果相同，则返回entry，否则继续下一轮查询。
```
python中冲突探测函数如下，其中j为当前的索引，perturb初始化为me_key的hash值，通过不断调整j和perturb值，会不断探测ma_table数组中的元素。由于perturb值是不断减小的，所以最终会退化为```j = 5 * j + 1```，假定元素数为8，初始```j = 0```， 使用退化后的探测函数探测到的索引依次是 ``` 0, 1, 6, 7, 4, 5, 2, 3, 0```。这个探测函数顺序有一定随机性，这其实是通过一个线性同余方程来获取```2**i```范围内的伪随机数作为索引，比线性探测函数如``` f(x) = x + 1```效果要好一些。其中```2**i```是哈希表大小。至于为什么 ```j = (5*j) + 1; j % 2**i;```这样能够得到```0...2**i-1```范围内的数字后再从头开始循环，证明方法暂时没有找到，不过简单验证确实是没有问题的。

```
 j = (5*j) + 1 + perturb; // perturb初始值为键的hash值
 perturb >>= PERTURB_SHIFT; // PERTURB_SHIFT=5，据说效果比较好
 use j % 2**i as the next table index;
```
根据这个冲突探测算法，可以看到字典中插入键值对时的冲突探测和解决流程。具体例子参见[http://www.laurentluce.com/posts/python-dictionary-implementation/](http://www.laurentluce.com/posts/python-dictionary-implementation/)
