
#Python源码剖析笔记1-Python对象初探
> 工作整两年了，用python最多，然而对于python内部机制不一定都清楚，每天沉醉于增删改查的简单逻辑编写，实在耗神。很多东西不用就忘记了，比如C语言，正好，python源码用C写的，分析python源码的同时又能温故C语言基础，实在是件很好的事情。另外，还有陈儒大神的《python源码剖析》做指引，分析也不至于没头没脑。期望在一个月的业余时间，能有所小成，以此为记。

## 1 python中的对象
python中，一切东西都是对象，在c语言实现中对应着结构体。首先当然还是从python内建对象开始看起，最基本的是PyIntObject, PyStringObject, PyListObject, PyDictObject这几个，他们分别属于int，string, list, dict类型。从python2.2之后有了new style class之后，这些内置对象都是继承自object类型，object在代码中对应PyBaseObject_Type。比如我们赋值语句a=3，那么a就是一个PyIntObject对象，它的类型是int,在代码中对应PyInt_Type，PyInt_Type也是一种对象，我们称之为类型对象。那么PyInt_Type它的类型是什么呢，答案是type， 对应到代码中就是PyType_Type。当然object也是一个类型对象，它的类型也是PyType_Type。这么一层层下去，PyType_Type也是个对象，那它的类型又是什么呢，没错，答案就是它的类型就是它自己，。看下面的验证代码：

```
##内建对象测试
In [1]: a = 3

In [2]: type(a)
Out[2]: int

In [3]: type(int)
Out[3]: type

In [4]: type(type)
Out[4]: type

In [5]: int.__base__
Out[5]: object

In [6]: type(object)
Out[6]: type
```

先分析下几个基础内建对象在C语言中的结构体以及常用的几个宏，为了方便，我用的也是陈儒大神分析的那个版本一致，版本是2.5.6.源码官网有下载。

```
// 内建对象基础
#define PyObject_HEAD                   \
        Py_ssize_t ob_refcnt;           \
        struct _typeobject *ob_type;

#define PyObject_HEAD_INIT(type)        \
        1, type,

#define PyObject_VAR_HEAD               \
        PyObject_HEAD                   \
        Py_ssize_t ob_size; /* Number of items in variable part */
#define Py_INVALID_SIZE (Py_ssize_t)-1

typedef struct _object {
        PyObject_HEAD
} PyObject;

typedef struct {
        PyObject_VAR_HEAD
} PyVarObject;

typedef struct _typeobject {
        PyObject_VAR_HEAD
        const char *tp_name; /* For printing, in format "<module>.<name>" */
        Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */
        destructor tp_dealloc;
        printfunc tp_print;
        getattrfunc tp_getattr;
        setattrfunc tp_setattr;
        cmpfunc tp_compare;
        reprfunc tp_repr;
        ...
} PyTypeObject;

typedef struct {
    PyObject_HEAD
    long ob_ival;
} PyIntObject;

typedef struct {
    PyObject_VAR_HEAD
    long ob_shash;
    int ob_sstate;
    char ob_sval[1];
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

如代码中所示，**PyObject是所有Python对象的基石，所有后续看到的对象都有一个相同的PyObject头部,从而我们可以在源码中看到所有的对象都可以用PyObject*指针指向**，这就是面向对象中经常用到的多态的技巧了。Python内部各个函数对象间也是通过PyObject*传递，即便本身这是一个PyIntObject类型的对象，代码中并不会用PyIntObject*指针进行传递，这也是为了实现多态。比如下面的函数：

```
void Print(PyObject* object) {
	object->ob_type->tp_print(object);
}
```

另外如代码中注释所说的，变长对象的ob_size指的是元素个数，不是字节数目。

##2 python对象引用计数
下面是几个常用的操作对象引用计数的宏定义(object.h)，一并列出，这里去除了一些调试时用的代码，更容易看明白代码含义。Py_NewReference是初始化时对象时设置引用计数, Py_INCREF和Py_DECREF分别用来增加引用技术和减少引用计数。从代码中可以看到，python增加引用和减少引用都是通过这些宏操作的，**有一点需要注意的是，当对象引用ob_refcnt减小到0时，会调用对象的析构函数，析构函数并不一定会调用free释放内存空间，因为频繁申请和释放内存严重影响性能，所以在后面看到python有大量用到内存池技术，对提升性能有很大效果。

**需要说明的是，类型对象是不在引用计数规则之中的，每个对象指向类型对象的指针并不视为类型对象的引用，也就是说不会影响类型对象的引用计数，类型对象永远不会被析构。**

```
#define _Py_NewReference(op) ((op)->ob_refcnt = 1)


#define _Py_Dealloc(op) (*(op)->ob_type->tp_dealloc)((PyObject *)(op)))

#define Py_INCREF(op) ((op)->ob_refcnt++)

#define Py_DECREF(op)                                   \
        if (--(op)->ob_refcnt != 0)                     \
        	;
        else                                            \
            _Py_Dealloc((PyObject *)(op))
            
#define Py_CLEAR(op)                            \
        do {                                    \
                if (op) {                       \
                        PyObject *tmp = (PyObject *)(op);       \
                        (op) = NULL;            \
                        Py_DECREF(tmp);         \
                }                               \
        } while (0)

/* Macros to use in case the object pointer may be NULL: */
#define Py_XINCREF(op) if ((op) == NULL) ; else Py_INCREF(op)
#define Py_XDECREF(op) if ((op) == NULL) ; else Py_DECREF(op)

```

##3 Python对象分类
python中的对象大致可以分为下面几类：
- 数值对象：如integer,float,boolean
- 序列集合对象：如string,list,tuple
- 字典对象：如dict
- 类型对象：如type
- 内部对象：如后面会看到的code，function，frame，module以及method对象等。

##4 参考资料
- 《python源码剖析》
