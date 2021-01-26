# Python虚拟机中的类机制

在python中，“一切皆对象”，包括面向对象概念的中的类在python也是对象。python中的对象存在两种关系——is-kind-of与is-instance-of。

+ is-kind-of：这种关系对应于面向对象中的基类与子类的关系；
+ is-instance-of：这种关系对应于面向对象中的类与实例之间的关系。

~~~python
class A(object):
    pass
a = A()
~~~

其中包含了三个对象：object（class A对象的基类）、class A对象以及a对象。这三者之间的关系也很明显：A与object之间存在is-kind-of关系，a与A以及a与object之间存在is-instance-of关系，即a是A以及object的一个实例。

python提供了一些方法以及属性来探测这些关系。

\__class__属性，type方法以及isinstance方法：探测两个对象之间的is-instance-of关系；

\__base__属性以及issubclass方法：探测两个对象之间的is-kind-of关系。

~~~python
class A(object):
    pass
a = A()

print(a.__class__)
print(isinstance(a,A))
print(isinstance(a,object))
print(A.__class__)
print(A.__base__)
print(isinstance(A,type))
print(issubclass(A,object))
print(a.__base__)
##输出
<class '__main__.A'>
True
True
<class 'type'>
<class 'object'>
True
True
Traceback (most recent call last):
  File "C:\Users\Administrator\Documents\test.py", line 11, in <module>
    print(a.__base__)
AttributeError: 'A' object has no attribute '__base__'
~~~

最后一条print语句抛出了异常。这也很好理解is-kind-of是类对象之间才有的关系，a是一个实例对象，不应该有表示is-kind-of关系的属性\__base__。

![image-20210119211742084](Python虚拟机中的类机制.assets/image-20210119211742084.png)

在python中有两个比较重要的类型对象——type对象以及object对象。首先，python中的type是一种特殊的对象，这种对象能够成为其他类型对象的类型，包括object对象以及type对象自身；其次，python中 除了object对象以外的所有类型对象都是直接或间接继承自object对现象，object对象没有基类。python中将type对象称为元类metaclass对象。

~~~pyhton
>>> type
<class 'type'>
>>> type.__base__
<class 'object'>
>>> type.__class__
<class 'type'>
>>> object
<class 'object'>
>>> object.__base__
>>> object.__class__
<class 'type'>
~~~

## 类型系统的初始化

PyType_Ready是python中类型初始化的起点。python中的内置类型对应的类型对象在python启动后已经作为全局对象存在，需要的仅仅是完善；而用户自定义类型的类型对象并不存在，需要申请内存，并创建初始化整个序列。对于内置类型来说，初始化就仅剩下PyType_Ready，而对于用户自定义类型来说，PyType_Ready仅仅是很小的一部分。

### 处理基类和type信息

我们从PyType_Type的初始化来考察PyType_Ready函数。

~~~C
int
PyType_Ready(PyTypeObject *type)//在这里，参数type是type对象的class对象（PyType_Type）
{
    PyObject *dict, *bases;
    PyTypeObject *base;
    Py_ssize_t i, n;
    
    /* Initialize tp_base (defaults to BaseObject unless that's us) */
    base = type->tp_base;//[A]
    if (base == NULL && type != &PyBaseObject_Type) {//如果type没有指定基类（base==NULL）且type不是Object的class对象（PyBaseObject_Type），则将type的基类设置为object对象。
        base = type->tp_base = &PyBaseObject_Type;
        Py_INCREF(base);
    }
    /* Initialize the base class *///[B]如果基类没有初始化，则先初始化基类
    if (base != NULL && base->tp_dict == NULL) {
        if (PyType_Ready(base) < 0)
            goto error;
    }
    /* Initialize ob_type if NULL.      This means extensions that want to be
       compilable separately on Windows can call PyType_Ready() instead of
       initializing the ob_type field of their type objects. */
    /* The test for base != NULL is really unnecessary, since base is only
       NULL when type is &PyBaseObject_Type, and we know its ob_type is
       not NULL (it's initialized to &PyType_Type).      But coverity doesn't
       know that. */
    if (Py_TYPE(type) == NULL && base != NULL)//[C]设置type信息
        Py_TYPE(type) = Py_TYPE(base);
    /*...*/
}
~~~

首先在代码[A]处，python虚拟机尝试获得待初始化的type的基类。这个信息是在TyTypeObject.tp_base中指定的。对于指定了tp_base的内置class对象，python就是用指定的类型作为class对象的基类；而对于tp_base指针为空的class对象，python虚拟机就是用PyBaseObject_Type（object对象）作为其基类。所以从这里可以看出，python中的所有class对象都是直接或者间接以object作为基类的（除了object对象，object对象没有基类）。

得到了基类之后，代码[B]处就要判断基类是否已经初始化了，如果没有，则需要对基类进行初始化。可以看到，判断基类初始化是否完成的条件是base->tp_dict是否为NULL。

最后在[C]处对class对象的ob_type信息进行了设置。实际上这个ob_type信息也就是对象的\__class__返回的信息。更进一步说，这里的ob_type就是元类（metaclass）。实际上，Python虚拟机就是将基类的metaclass作为了子类的metalcass。这里对于PyType_Type来说，其metaclass正是object的metaclass。而在PyBaseObject\_Type的定义中我们可以看到其ob_type被设置成了PyType_Type，这正好与上面的python对象关系图互相印证。

### 处理基类列表

~~~C
/*...*/
if (Py_TYPE(type) == NULL && base != NULL)
        Py_TYPE(type) = Py_TYPE(base);

    /* Initialize tp_bases */
    bases = type->tp_bases;
    if (bases == NULL) {//如果bases为空，则根据情况设置基类列表
        if (base == NULL)//如果base为空，则该class对象的基类列表为空的tuple
            bases = PyTuple_New(0);
        else//如果base不空，则说明基类列表中至少包含一个base对象
            bases = PyTuple_Pack(1, base);
        if (bases == NULL)
            goto error;
        type->tp_bases = bases;
    }
/*...*/
~~~

在这里，python虚拟机将处理类型的基类列表，因为python支持多重继承，所以每一个class对象都会有一个基类列表。

### 填充tp_dict

~~~C
/*...*/
/* Initialize tp_dict */
    dict = type->tp_dict;
    if (dict == NULL) {
        dict = PyDict_New();
        if (dict == NULL)
            goto error;
        type->tp_dict = dict;
    }

    /* Add type-specific descriptors to tp_dict */
    if (add_operators(type) < 0)
        goto error;
    if (type->tp_methods != NULL) {
        if (add_methods(type, type->tp_methods) < 0)
            goto error;
    }
    if (type->tp_members != NULL) {
        if (add_members(type, type->tp_members) < 0)
            goto error;
    }
    if (type->tp_getset != NULL) {
        if (add_getset(type, type->tp_getset) < 0)
            goto error;
    }
/*...*/
~~~

在这个阶段，完成了将("\__add__",&nb_add)加入tp\_dict的过程。这个阶段的add_operators、add_methods、add_members以及add_getset都是完成这样的填充过程的。那么，python是如何找到操作名称与具体的实现动作之间的关系的呢？答案是python源码中已经预先定义好了，存放在一个名为slotdefs的全局数组之中。

#### slot与操作排序

在python内部，slot可以视为标识PyTypeObject中定义的操作，一个操作对应一个slot，但是slot不仅仅包含一个函数指针，还包含其他一些信息。在python内部，slot是通过slotdefs这个结构体来实现的。

~~~C
typedef struct wrapperbase slotdef;

struct wrapperbase {
    const char *name;
    int offset;
    void *function;
    wrapperfunc wrapper;
    const char *doc;
    int flags;
    PyObject *name_strobj;
};
~~~

在一个slot中，存储着与PyTypeObject中一种操作对应的各种信息。比如，name是操作的名称，比如字符串"\__add__"；offset则是操作的函数地址在PyHeapTypeObject中的偏移量；而function则指向一种被称为slot function的函数。

python中提供了多个宏来定义一个slot，其中最基本的是TPSLOT和ETSLOT。

~~~C
#define TPSLOT(NAME, SLOT, FUNCTION, WRAPPER, DOC) \
    {NAME, offsetof(PyTypeObject, SLOT), (void *)(FUNCTION), WRAPPER, \
     PyDoc_STR(DOC)}

#define ETSLOT(NAME, SLOT, FUNCTION, WRAPPER, DOC) \
    {NAME, offsetof(PyHeapTypeObject, SLOT), (void *)(FUNCTION), WRAPPER, \
     PyDoc_STR(DOC)}
~~~

TPSLOT与EPSLOT的区别在于TPSLOT计算的是操作对应的函数指针（比如nb_add）在PyTypeObject中的偏移量，而ETSLOT计算的是函数指针在PyHeapTypeObject中的偏移量。但是通过PyHeapTypeObject的代码可以发现，PyHeapTypeObject的第一个域就是PyTypeObject，所以TPSLOT计算出的偏移量实际上也就是相对于PyHeapTypeObject的偏移量。

~~~C
/* The *real* layout of a type object when allocated on the heap */
typedef struct _heaptypeobject {
    /* Note: there's a dependency on the order of these members
       in slotptr() in typeobject.c . */
    PyTypeObject ht_type;
    PyAsyncMethods as_async;
    PyNumberMethods as_number;
    PyMappingMethods as_mapping;
    PySequenceMethods as_sequence; /* as_sequence comes after as_mapping,
                                      so that the mapping wins when both
                                      the mapping and the sequence define
                                      a given operator (e.g. __getitem__).
                                      see add_operators() in typeobject.c . */
    PyBufferProcs as_buffer;
    PyObject *ht_name, *ht_slots, *ht_qualname;
    struct _dictkeysobject *ht_cached_keys;
    /* here are optional user slots, followed by the members. */
} PyHeapTypeObject;
~~~

对于一个PyTypeObject来说，有的操作，比如nb_add，其函数指针是放在PyNumberMethods中存储的，而在PyTypeObject中却是通过一个tp_as_number指针指向另一个PyNumberMethods结构，所以实际上根本没有办法计算出nb_add在PyTypeObject中的偏移量，只能计算其在PyHeapTypeObject中的偏移量。

通过以上描述可以发现，通过offset是无法得到PyLongObject中为long准备的nb_add的。所以这里的offset另有他用。

offset的作用到底是什么呢？答案是对操作进行排序。为了理解为什么需要对操作进行排序，需要来看python预定义的slot集合——slotdefs。

~~~C
static slotdef slotdefs[] = {
    /*...*/
    BINSLOT("__add__", nb_add, slot_nb_add,
           "+"),
    RBINSLOT("__radd__", nb_add, slot_nb_add,
           "+"),
    /*...*/
    SQSLOT("__len__", sq_length, slot_sq_length, wrap_lenfunc,
           "__len__($self, /)\n--\n\nReturn len(self)."),
    /*...*/
    MPSLOT("__getitem__", mp_subscript, slot_mp_subscript,
           wrap_binaryfunc,
           "__getitem__($self, key, /)\n--\n\nReturn self[key]."),
    /*...*/
    SQSLOT("__getitem__", sq_item, slot_sq_item, wrap_sq_item,
           "__getitem__($self, key, /)\n--\n\nReturn self[key]."),
    /*...*/
};
~~~

在slotdefs中，存在多个操作对应同一个操作名的情况，同样也存在着同一个操作对应不同操作名的情况。对于相同操作对应不同操作名的情况，在填充tp_dict时就会出现到底填充哪一个操作的问题。

为了解决这个问题，就需要利用slot中的offset信息对slot进行排序。