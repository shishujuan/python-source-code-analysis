# Python源码剖析笔记2-Python整数对象

> 千里之行始于足下，从简单的类别开始分析，由浅入深也不至于自己丧失信心。先来看看Python整数对象，也就是python中的PyIntObject对象，对应的类型对象是PyInt_Type。

## 1 Python整数对象概览
为了性能考虑，python中对小整数有专门的缓存池，这样就不需要每次使用小整数对象时去用malloc分配内存以及free释放内存。python2.5.6中默认小整数的范围为[-5, 257)，你也可以修改这个范围并重新编译Python。

小整数有缓存池，那么小整数之外的大整数怎么避免重复分配和回收内存呢？Python的方案是PyIntBlock。PyIntBlock这个结构就是一块内存，里面保存PyIntObject对象。一个PyIntBlock默认存放N_INTOBJECTS对象，2.5.6版本在32位ubuntu中这个值为82.PyIntBlock链表通过block_list维护，每个block中都维护一个PyIntObject数组objects，block的objects可能会有些内存空闲，因此需要另外用一个free_list链表串起来这些空闲的项以方便再次使用。

小整数的缓存池最终实现也是生存在block_list维护的内存上，在python初始化时，会调用_PyInt_Init函数申请内存并创建小整数对象。

## 2 PyIntObject和PyInt_Type
下面看看PyIntObject和PyInt_Type的定义(在源文件Includes/intobject.h和Objects/intobject.c)。

```
//先补上PyObject_HEAD

#define PyObject_HEAD                   \
        Py_ssize_t ob_refcnt;           \
        struct _typeobject *ob_type;

#define PyObject_HEAD_INIT(type)        \
        1, type,

typedef struct {
    PyObject_HEAD
    long ob_ival;
} PyIntObject;

```
从定义中可以看到，在32位系统中，一个PyIntObject结构体所占大小为12个字节，这个也可以通过python自带的模块来验证，运行 ```import sys; sys.getsizeof(3)```。

PyInt_Type是PyType_Object类型的对象，它的定义在Objects/intobject.c中。

```
//PyTypeObject定义在Includes/object.h中

PyTypeObject PyInt_Type = {
    PyObject_HEAD_INIT(&PyType_Type) //初始化对象头部
    0,                               //因为PyTypeObject是个PyVarObject对象，因此这里需要设置下大小为0.
    "int",                           //用来打印的字段，比如我们type(3)返回的int字符串就是来自这里。
    sizeof(PyIntObject),             //对象基本大小
    0,                               //如果是可变大小对象，这个字段是对象里面存储项的大小
    (destructor)int_dealloc,        /* tp_dealloc */
    (printfunc)int_print,           /* tp_print */
    0,                  /* tp_getattr */
    0,                  /* tp_setattr */
    (cmpfunc)int_compare,           /* tp_compare */
    (reprfunc)int_repr,         /* tp_repr */
    &int_as_number,             /* tp_as_number */
    0,                  /* tp_as_sequence */
    0,                  /* tp_as_mapping */
    (hashfunc)int_hash,         /* tp_hash */
        0,                  /* tp_call */
        (reprfunc)int_repr,         /* tp_str */
    PyObject_GenericGetAttr,        /* tp_getattro */
    0,                  /* tp_setattro */
    0,                  /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_CHECKTYPES |
        Py_TPFLAGS_BASETYPE,        /* tp_flags */
    int_doc,                /* tp_doc */
    0,                  /* tp_traverse */
    0,                  /* tp_clear */
    0,                  /* tp_richcompare */
    0,                  /* tp_weaklistoffset */
    0,                  /* tp_iter */
    0,                  /* tp_iternext */
    int_methods,                /* tp_methods */
    0,                  /* tp_members */
    0,                  /* tp_getset */
    0,                  /* tp_base */
    0,                  /* tp_dict */
    0,                  /* tp_descr_get */
    0,                  /* tp_descr_set */
    0,                  /* tp_dictoffset */
    0,                  /* tp_init */
    0,                  /* tp_alloc */
    int_new,                /* tp_new */
    (freefunc)int_free,                 /* tp_free */
};

//Objects/intobject.c中定义了PyIntObject支持的数值操作，一共39个。
static PyNumberMethods int_as_number = {
    (binaryfunc)int_add,    /*nb_add*/
    (binaryfunc)int_sub,    /*nb_subtract*/
    (binaryfunc)int_mul,    /*nb_multiply*/
    (binaryfunc)int_classic_div, /*nb_divide*/
    (binaryfunc)int_mod,    /*nb_remainder*/
    (binaryfunc)int_divmod, /*nb_divmod*/
    ...
};
```
看了PyInt_Type定义，发现主要就是对象头部初始化以及对一些函数的初始化设置。如int_compare是PyIntObject对象的比较函数，而int_print是打印PyIntObject的函数等。此外，还有个很重要的初始化的地方是int_as_number，这个结构定义了一个对象作为数值对象时的操作信息。如int_add，int_sub等用来执行整数加减。

## 3 PyIntObject创建和相关函数
python中为PyIntObject对象创建主要提供了3个函数，如下所示。这三个函数分别从字符串，unicode对象以及long值生成PyIntObject对象。其中PyInt_FromUnicode最终调用PyInt_FromString，而PyInt_String最终调用PyInt_FromLong函数，因此我这里先简单分析下PyInt_FromLong函数。

```
//Includes/intobject.h中
(PyObject *) PyInt_FromString(char*, char**, int);
(PyObject *) PyInt_FromUnicode(Py_UNICODE*, Py_ssize_t, int);
(PyObject *) PyInt_FromLong(long);
```

```
PyObject *
PyInt_FromLong(long ival)
{
    register PyIntObject *v;
#if NSMALLNEGINTS + NSMALLPOSINTS > 0
    if (-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS) {
        v = small_ints[ival + NSMALLNEGINTS];
        Py_INCREF(v);
#ifdef COUNT_ALLOCS
        if (ival >= 0)
            quick_int_allocs++;
        else
            quick_neg_int_allocs++;
#endif
        return (PyObject *) v;
    }
#endif
    if (free_list == NULL) { //PyIntBlock的objects没有空闲空间或者第一次分配时，调用fill_free_list函数分配PyIntBlock
        if ((free_list = fill_free_list()) == NULL)
            return NULL;
    }
    /* Inline PyObject_New */
    v = free_list;
    free_list = (PyIntObject *)v->ob_type; //更新free_list,指向PyIntBlock的objects的下一个对象
    PyObject_INIT(v, &PyInt_Type);
    v->ob_ival = ival;
    return (PyObject *) v;
}

```
这里涉及到小整数和PyIntBlock相关内容，在创建PyIntObject对象的时候，会首先判断数值大小是否在小整数范围内，如果在，则直接从小整数对象池small_ints中取。其中small_ints是在python初始化的时候调用_PyInt_Init函数创建的，代码如下：

```
#define BLOCK_SIZE  1000    /* 1K less typical malloc overhead */
#define BHEAD_SIZE  8   /* Enough for a 64-bit pointer */
#define N_INTOBJECTS    ((BLOCK_SIZE - BHEAD_SIZE) / sizeof(PyIntObject))

struct _intblock {
    struct _intblock *next;
    PyIntObject objects[N_INTOBJECTS];
};

typedef struct _intblock PyIntBlock;

static PyIntBlock *block_list = NULL;
static PyIntObject *free_list = NULL;

static PyIntObject *
fill_free_list(void)
{
    PyIntObject *p, *q;
    /* Python's object allocator isn't appropriate for large blocks. */
    p = (PyIntObject *) PyMem_MALLOC(sizeof(PyIntBlock));
    if (p == NULL)
        return (PyIntObject *) PyErr_NoMemory();

     /*通过next指针链接PyIntBlock，最新分配的插入到链表头部，block_list总是指向最新分配的PyIntBlock*/
    ((PyIntBlock *)p)->next = block_list;
    block_list = (PyIntBlock *)p;

    /* Link the int objects together, from rear to front, then return
       the address of the last int object in the block.
       将PyIntBlock的objects数组转换为单向链表，从后往前通过ob_type连接，并返回最后一个对象。
        */
    p = &((PyIntBlock *)p)->objects[0];
    q = p + N_INTOBJECTS;
    while (--q > p)
        q->ob_type = (struct _typeobject *)(q-1);
    q->ob_type = NULL;
    return p + N_INTOBJECTS - 1;
}

int
_PyInt_Init(void)
{
    PyIntObject *v;
    int ival;
#if NSMALLNEGINTS + NSMALLPOSINTS > 0
    for (ival = -NSMALLNEGINTS; ival < NSMALLPOSINTS; ival++) {
              if (!free_list && (free_list = fill_free_list()) == NULL)
            return 0;
        /* PyObject_New is inlined */
        v = free_list;
        free_list = (PyIntObject *)v->ob_type; // small_ints数组存储小整数对象，注意下small_ints[0]存储的是-5,small_ints[5]存储的才是0.free_list从后往前通过ob_type链接，每次都是从后面取。
        PyObject_INIT(v, &PyInt_Type);
        v->ob_ival = ival;
        small_ints[ival + NSMALLNEGINTS] = v;
    }
#endif
    return 1;
}

static void
int_dealloc(PyIntObject *v)
{
    if (PyInt_CheckExact(v)) {
        v->ob_type = (struct _typeobject *)free_list;
        free_list = v; //修改free_list
    }
    else
        v->ob_type->tp_free((PyObject *)v);
}


```

## 4 调试PyIntObject
按照《Python源码剖析》中的方法，可以打印出一些调试信息。先是修改int_print函数对象，来打印小整数对象和block_list,free_list等相关信息。修改Objects/intobject.c中的int_print函数，代码如下：

```
static int values[10];static PyIntObject** addr[10];static int refcounts[10];/* ARGSUSED */static intint_print(PyIntObject *v, FILE *fp, int flags)     /* flags -- not used but required by interface */{	//fprintf(fp, "%ld", v->ob_ival);	PyIntObject *intObjectPtr;	PyIntBlock *p = block_list;	PyIntBlock *last = NULL;	int count = 0;	int i;	while (p != NULL) 	{		++count;		last = p;		p = p->next;	}	intObjectPtr = last->objects;	intObjectPtr += N_INTOBJECTS - 1;	printf(" address @%p, value=0x%x\n", v, v->ob_ival);	for (i=0; i<10; ++i, --intObjectPtr)	{		values[i] = intObjectPtr->ob_ival;		refcounts[i] = intObjectPtr->ob_refcnt;		addr[i] = intObjectPtr;	}	printf(" value: ");	for (i=0; i < 8; ++i)	{		printf("%d:%p\t", values[i], addr[i]);	}	printf("\n");	printf(" refcnt: ");	for (i=0; i < 8; ++i)	{		printf("%d\t", refcounts[i]);	}	printf("\n");	printf("block_list count: %d\n", count);	printf("free_list: %p\n", free_list);	return 0;}
```

编译源码后运行python，测试的结果如下:

```
>>> a = -12345>>> a address @0x9415790, value=0xffffcfc7 value: -5:0x9414f80	-4:0x9414f74	-3:0x9414f68	-2:0x9414f5c	-1:0x9414f50	0:0x9414f44	1:0x9414f38	2:0x9414f2c	 refcnt: 1	1	2	1	32	109	49	30	block_list count: 4free_list: 0x941579c>>> b = -12345>>> b address @0x941579c, value=0xffffcfc7 value: -5:0x9414f80	-4:0x9414f74	-3:0x9414f68	-2:0x9414f5c	-1:0x9414f50	0:0x9414f44	1:0x9414f38	2:0x9414f2c	 refcnt: 1	1	2	1	32	109	49	30	block_list count: 4free_list: 0x94157b4>>> c = -5>>> c address @0x9414f80, value=0xfffffffb value: -5:0x9414f80	-4:0x9414f74	-3:0x9414f68	-2:0x9414f5c	-1:0x9414f50	0:0x9414f44	1:0x9414f38	2:0x9414f2c	 refcnt: 5	1	2	1	32	109	49	30	block_list count: 4free_list: 0x94157b4>>> d = -5>>> d address @0x9414f80, value=0xfffffffb value: -5:0x9414f80	-4:0x9414f74	-3:0x9414f68	-2:0x9414f5c	-1:0x9414f50	0:0x9414f44	1:0x9414f38	2:0x9414f2c	 refcnt: 6	1	2	1	32	109	49	30	block_list count: 4free_list: 0x94157b4
```

通过测试可以发现，a和b虽然值都是-12345，但是他们是两个不同的对象，地址也不一样。c和d都是-5，但是由于小整数缓存机制，所以c和d的地址是一样，是同一个对象。同时我们可以观察到小整数中-5到2这8个整数的地址是从高到低的，相隔12个字节，这也就验证了objects数组是从后往前通过ob_type字段连接成链表的。

另外，我们可以用id(xxx)来获取对象的地址(当然这个地址是指逻辑地址)，比如上面例子中的id(d)的结果就0x9414f80。id对应的代码在Python/bltinmodule.c的builtin_id(PyObject *self, PyObject *v)函数，其功能就是打印出python对象的地址。同理，id(int)就是PyInt_Type类型对象的地址。

## 5 总结

简单总结下，Python维护多个PyIntBlock对象，一个PyIntBlock中存储多个整数。PyIntBlock之间通过链表连接,最新分配的PyIntBlock加入在链表首部，block_list为链表首部。而PyIntBlock中的整数对象数组objects通过ob_type指针从后往前链接，freelist为该链表首部，即objects数组的最后一个对象。

整数对象引用减少到0时，调用int_dealloc函数释放对象。需要注意的是，PyIntObject释放对象的时候，并不释放内存，只是将这块内存作为可用内存加入到free_list中，并将free_list指向刚刚释放的对象。

