### python 对象的分类

根据对象是否可变以及是都定长，python 中的对象可以分为四类：

| 类别       | 特点                     |
| ---------- | ------------------------ |
| 可变对象   | 对象创建后，可以被修改； |
| 不可变对象 | 对象创建后，不能被修改； |
| 定长对象   | 对象大小固定；           |
| 变长对象   | 对象大小不固定；         |

### PyObject

这是一个结构体，它是所有 python 对象的基石，定义如下：

```c
// Include/object.h
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
} PyObject;
```

它共有 3 个字段：

- _PyObject_HEAD_EXTRA 是一个宏定义，定义如下：

  ```c
  // Include/object.h
  #ifdef Py_TRACE_REFS
  /* Define pointers to support a doubly-linked list of all live heap objects. */
  #define _PyObject_HEAD_EXTRA            \
      struct _object *_ob_next;           \
      struct _object *_ob_prev;
  
  #define _PyObject_EXTRA_INIT 0, 0,
  
  #else
  #define _PyObject_HEAD_EXTRA
  #define _PyObject_EXTRA_INIT
  #endif
  ```

  它将每一个对象串成一个双向链表，不过默认这部分都为空；

- ob_refcnt, 这个即所谓的引用计数，标记该对象目前被引用的总数，方便垃圾回收；

- ob_type, 这是一个比较复杂的结构体，通过这个字段，就能知道当前对象所属的类型、拥有哪些方法等；



### PyVarObject

PyObject 只能用来标记定长对象，而 PyVarObject 则用来表示变长对象：

```c
// Include/object.h
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;
```

这个结构体增加了一个 ob_size 字段，用来表示可变对象拥有的元素个数。



### 内建类型表示

内建对象根据自己类型的忒点，选择 PyObject 和 PyVarObject 作为自己的头部；源码中声明了两个宏定义，方便使用：

```c
#define PyObject_HEAD          PyObject ob_base;
#define PyObject_VAR_HEAD      PyVarObject ob_base;
```



#### PyFloatObject

浮点数的结构体定义如下：

```c
// Include/floatobject.h
typedef struct {
    PyObject_HEAD
    double ob_fval;
} PyFloatObject;
```

python 中的浮点数是个定长对象，因此它以 PyObject  作为头，再在它后边加一个 c 原生的 float 类型作为实际值。



#### PyListObject

列表的结构体定义如下：

```c
// Include/listobject.h
typedef struct {
    PyObject_VAR_HEAD
    /* Vector of pointers to list elements.  list[0] is ob_item[0], etc. */
    PyObject **ob_item;

    /* ob_item contains space for 'allocated' elements.  The number
     * currently in use is ob_size.
     * Invariants:
     *     0 <= ob_size <= allocated
     *     len(list) == ob_size
     *     ob_item == NULL implies ob_size == allocated == 0
     * list.sort() temporarily sets allocated to -1 to detect mutations.
     *
     * Items must normally not be NULL, except during construction when
     * the list is not yet visible outside the function that builds it.
     */
    Py_ssize_t allocated;
} PyListObject;
```

list 是可变对象，它以 PyVarObject 作为头，并增加了 ob_item 指向实际的列表元素内存区域，而 allocated 则表示分配的多少个元素的空间。



### PyTypeObject

上面给定的两个结构体中，有一个 ob_type 字段，它会指向该对象的类型，以解决以下两个问题：

1. 对象的内存信息；
2. 对象的可执行操作；



它的定义如下：

```c
// Include/object.h
typedef struct _typeobject {
    PyObject_VAR_HEAD
    const char *tp_name; /* For printing, in format "<module>.<name>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */

    /* Methods to implement standard operations */

    destructor tp_dealloc;
    printfunc tp_print;
    getattrfunc tp_getattr;
    setattrfunc tp_setattr;
    PyAsyncMethods *tp_as_async; /* formerly known as tp_compare (Python 2)
                                    or tp_reserved (Python 3) */
    reprfunc tp_repr;

    /* Method suites for standard classes */

    PyNumberMethods *tp_as_number;
    PySequenceMethods *tp_as_sequence;
    PyMappingMethods *tp_as_mapping;

    /* More standard operations (here for binary compatibility) */

    hashfunc tp_hash;
    ternaryfunc tp_call;
    reprfunc tp_str;
    getattrofunc tp_getattro;
    setattrofunc tp_setattro;

    /* Functions to access object as input/output buffer */
    PyBufferProcs *tp_as_buffer;

    /* Flags to define presence of optional/expanded features */
    unsigned long tp_flags;

    const char *tp_doc; /* Documentation string */

    /* Assigned meaning in release 2.0 */
    /* call function for all accessible objects */
    traverseproc tp_traverse;

    /* delete references to contained objects */
    inquiry tp_clear;

    /* Assigned meaning in release 2.1 */
    /* rich comparisons */
    richcmpfunc tp_richcompare;

    /* weak reference enabler */
    Py_ssize_t tp_weaklistoffset;

    /* Iterators */
    getiterfunc tp_iter;
    iternextfunc tp_iternext;

    /* Attribute descriptor and subclassing stuff */
    struct PyMethodDef *tp_methods;
    struct PyMemberDef *tp_members;
    struct PyGetSetDef *tp_getset;
    struct _typeobject *tp_base;
    PyObject *tp_dict;
    descrgetfunc tp_descr_get;
    descrsetfunc tp_descr_set;
    Py_ssize_t tp_dictoffset;
    initproc tp_init;
    allocfunc tp_alloc;
    newfunc tp_new;
    freefunc tp_free; /* Low-level free-memory routine */
    inquiry tp_is_gc; /* For PyObject_IS_GC */
    PyObject *tp_bases;
    PyObject *tp_mro; /* method resolution order */
    PyObject *tp_cache;
    PyObject *tp_subclasses;
    PyObject *tp_weaklist;
    destructor tp_del;

    /* Type attribute cache version tag. Added in version 2.6 */
    unsigned int tp_version_tag;

    destructor tp_finalize;

#ifdef COUNT_ALLOCS
    /* these must be last and never explicitly initialized */
    Py_ssize_t tp_allocs;
    Py_ssize_t tp_frees;
    Py_ssize_t tp_maxalloc;
    struct _typeobject *tp_prev;
    struct _typeobject *tp_next;
#endif
} PyTypeObject;
```

这是一个可变对象，有一些比较重要的字段：

- tp_name，表示类型名称；
- tp_base, 指向基类对象信息，不知道怎么处理多继承问题；
- tp_basicsize tp_itemsize，创建对象所需的内存信息；
- 各种操作函数指针；

每一个实例对象都会通过 ob_type 指向自己的类信息，而每个类也通过自己的 ob_type 指向自己的类型信息；在 python 中，每个类在声明时，都会创建一个全局唯一的类对象，该类的各个实例通过指针引用到这个对象，而这个对象又通过自己的 ob_type 指向全局唯一的 type 实例。以浮点数为例：

```c
// Include/floatobject.h
PyTypeObject PyFloat_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "float",
    sizeof(PyFloatObject),
    ...
    0,                                          /* tp_base */
    ...
};

// Include/typeobject.h
PyTypeObject PyType_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "type",                                     /* tp_name */
    sizeof(PyHeapTypeObject),                   /* tp_basicsize */
    sizeof(PyMemberDef),                        /* tp_itemsize */
    ...
    0,                                          /* tp_base */
    ...
}
```

从上面可以看到，浮点数类型在初始化时，会构建一个 PyFloat_Type 变量，它初始化了所有浮点数类型的可执行方法，并且将自己的 ob_type 字段指向全局唯一的 PyType_Type 变量，而这个变量在初始化时则将自己的ob_type 指向自己。它就是 python 中所谓的元类型。



### PyBaseObject_Type

python 中一切皆对象，不管是 list 还是 float 还是其它类型，他们的基类都是 object，而 type 也是继承于 object 的。每个类型实例都通过自己的 tp_base 指向自己的基类对象，但是可以看到上面的 PyFloat_Type 和 PyType_Type 在初始化时，它们的 tp_base 居然都初始化成了 null。

目前看，python 中还有一个奇怪的函数，叫 _Py_ReadyTypes：

```c
// Objects/object.c
void _Py_ReadyTypes(void){
    ...
    if (PyType_Ready(&PyFloat_Type) < 0)
        Py_FatalError("Can't initialize float type");
    ...
}
```

这个函数会调用一个叫 PyType_Ready 的函数，将每个类型变量进行额外的一些初始化工作，这其中就包括将类型变量的 tp_base 指向一个叫 PyBaseObject_Type 的变量，而它就是 object 这个所有类型的基类：

```c
// Objects/typeobject.c
int PyType_Ready(PyTypeObject *type)
{
    ...
	base = type->tp_base;
    if (base == NULL && type != &PyBaseObject_Type) {
        base = type->tp_base = &PyBaseObject_Type;
        Py_INCREF(base);
    }
    ...
}

// Python/pyfilecycle.c
PyTypeObject PyBaseObject_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "object",                                   /* tp_name */
    sizeof(PyObject),                           /* tp_basicsize */
    ...
    0,                                          /* tp_base */
    ...
};
```

而在 PyBaseObject_Type 初始化的过程中，它的 ob_type 指向了 PyType_Type, 而 tp_base 同样指向 null，并且永远都为 null，避免继承链形成循环。