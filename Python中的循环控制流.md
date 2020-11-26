# 循环控制流

### for循环控制流

在分析Python中的循环控制流之前首先来看一段循环代码。

```python
def a():
    ls = [1,2]
    for i in ls:
        print(i)
```
编译之后的python字节码。
```python
  2           0 LOAD_CONST               1 (1)
              2 LOAD_CONST               2 (2)
              4 BUILD_LIST               2
              6 STORE_FAST               0 (ls)

  3           8 LOAD_FAST                0 (ls)
             10 GET_ITER
        >>   12 FOR_ITER                12 (to 26)
             14 STORE_FAST               1 (i)

  4          16 LOAD_GLOBAL              0 (print)
             18 LOAD_FAST                1 (i)
             20 CALL_FUNCTION            1
             22 POP_TOP
             24 JUMP_ABSOLUTE           12
        >>   26 LOAD_CONST               0 (None)
             28 RETURN_VALUE
```

在python字节码中，8 load_fast对应的是for循环的开始。首先，虚拟机将被循环的对象（ls）压入堆栈，然后get_iter开启了整个for循环的过程。

#### 字节码get_iter

```C
case TARGET(GET_ITER): {
            /* before: [obj]; after [getiter(obj)] */
            PyObject *iterable = TOP();
            PyObject *iter = PyObject_GetIter(iterable);
            Py_DECREF(iterable);
            SET_TOP(iter);
            if (iter == NULL)
                goto error;
            PREDICT(FOR_ITER);
            PREDICT(CALL_FUNCTION);
            DISPATCH();
        }
```

首先，get_iter从栈顶获取带迭代对象。然后调用PyObject_GetIter获取该对象的迭代器。

PyObject_GetIter代码

```C
typedef PyObject *(*getiterfunc) (PyObject *);

PyObject *
PyObject_GetIter(PyObject *o)
{
    PyTypeObject *t = o->ob_type;
    getiterfunc f;

    f = t->tp_iter;
    if (f == NULL) {
        if (PySequence_Check(o))
            return PySeqIter_New(o);
        return type_error("'%.200s' object is not iterable", o);
    }
    else {
        PyObject *res = (*f)(o);//获得类型对象中的tp_iter操作
        if (res != NULL && !PyIter_Check(res)) {
            PyErr_Format(PyExc_TypeError,
                         "iter() returned non-iterator "
                         "of type '%.100s'",
                         res->ob_type->tp_name);
            Py_DECREF(res);
            res = NULL;
        }
        return res;
    }
}
```

显然，PyObject_GetIter是通过调用对象对应的类型对象的tp_iter操作来获得与对象关联的迭代器的。Python中的迭代器与java、C#或C++中的迭代器概念是一致的。Python中的迭代器也是实在的对象。

```C
typedef struct {
    PyObject_HEAD
    Py_ssize_t it_index;
    PyListObject *it_seq; /* Set to NULL when iterator is exhausted */
} listiterobject;
```

listiterobject是对list对象的简单封装。其中使用it_index记录当前迭代位置，it_seq记录被迭代的list对象。

我们来看一下list对象的type对象

```C
PyTypeObject PyList_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "list",
    sizeof(PyListObject),
    0,
    (destructor)list_dealloc,                   /* tp_dealloc */
        /*...*/
    list_iter,                                  /* tp_iter */
        /*...*/
};
```

list_type对象的tp_iter域指向了list_iter

```C
static PyObject *
list_iter(PyObject *seq)
{
    listiterobject *it;

    if (!PyList_Check(seq)) {
        PyErr_BadInternalCall();
        return NULL;
    }
    it = PyObject_GC_New(listiterobject, &PyListIter_Type);
    if (it == NULL)
        return NULL;
    it->it_index = 0;//记录当前list的迭代位置
    Py_INCREF(seq);
    it->it_seq = (PyListObject *)seq;//被迭代的list对象
    _PyObject_GC_TRACK(it);
    return (PyObject *)it;
}
```

get_iter获取了list对象的迭代器对象之后，通过set_top宏强制将其设置为栈顶元素。

#### 字节码for_iter

get_iter获取迭代器之后，for_iter实现对迭代器的迭代。for_iter的实现如下

```C
case TARGET(FOR_ITER): {
    PREDICTED(FOR_ITER);
    /* before: [iter]; after: [iter, iter()] *or* [] */
    PyObject *iter = TOP();
    PyObject *next = (*iter->ob_type->tp_iternext)(iter);
    if (next != NULL) {
        PUSH(next);
        PREDICT(STORE_FAST);
        PREDICT(UNPACK_SEQUENCE);
        DISPATCH();
    }
    if (_PyErr_Occurred(tstate)) {
        if (!_PyErr_ExceptionMatches(tstate, PyExc_StopIteration)) {
            goto error;
        }
        else if (tstate->c_tracefunc != NULL) {
            call_exc_trace(tstate->c_tracefunc, tstate->c_traceobj, tstate, f);
        }
        _PyErr_Clear(tstate);
    }
    /* iterator ended normally */
    STACK_SHRINK(1);
    Py_DECREF(iter);
    JUMPBY(oparg);
    PREDICT(POP_BLOCK);
    DISPATCH();
}
```

for_iter指令反复调用迭代器的tp_iternext方法进行迭代。该方法总是返回与迭代器关联的容器对象的下一个元素，如果当前已经抵达了迭代器的结束位置，那么返回NULL以表示迭代结束。

tp_iternext的具体实现如下

```C
static PyObject *
listiter_next(listiterobject *it)
{
    PyListObject *seq;
    PyObject *item;

    assert(it != NULL);
    seq = it->it_seq;
    if (seq == NULL)
        return NULL;
    assert(PyList_Check(seq));

    if (it->it_index < PyList_GET_SIZE(seq)) {//seq（被迭代的list对象）中依然还有可迭代的元素
        item = PyList_GET_ITEM(seq, it->it_index);
        ++it->it_index;//调整it_index使其指向下一个元素。
        Py_INCREF(item);
        return item;
    }

    it->it_seq = NULL;
    Py_DECREF(seq);
    return NULL;//迭代结束返回NULL
}
```

回到for_iter中的代码，如果next不为NULL，那么将next入栈，并执行指令预测。在本例中，虚拟机成功预测到了store_fast指令。store_fast以及接下来的三条指令（14-22）实现了对i的赋值以及输出（调用print函数）操作。

由于python中的函数必须要有一个返回值（默认返回None），所以print执行结束之后栈顶会存放对None对象的引用（pirnt返回None），指令pop_top的作用是弹出栈顶的None对象以保持堆栈平衡。

```C
case TARGET(POP_TOP): {
            PyObject *value = POP();
            Py_DECREF(value);
            FAST_DISPATCH();
        }
```

可以看到，字节码pop_top只是单纯将栈顶元素出栈并不进行其他操作。

指令jump_absolute指令实现了python虚拟机控制流的回退。

```C
case TARGET(JUMP_ABSOLUTE): {
            PREDICTED(JUMP_ABSOLUTE);
            JUMPTO(oparg);
    }
```

宏定义JUMPTO将next_instr设置为距离f->f_code->co_code oparg个字节的位置，在本例中此位置即为字节码for_iter。

for_iter字节码接受一个参数，这个参数指定了for循环的结束位置。在for_iter中如果返回的next为NULL则代表循环结束，此时进入跳出循环的流程。

首先，宏定义STACK_SHRINK(1)将栈顶元素（当前存放的是迭代完成的迭代器）弹出。

然后调用JUMP_BY宏实现了控制流的跳转。

```C
#define JUMPBY(x)       (next_instr += (x) / sizeof(_Py_CODEUNIT))
```

JUMPBY是实现相对跳转的宏，作用是将next_instr设置为距离当前指令oparg个字节的位置。在这里，虚拟机的控制流通过JUMP_BY宏跳转到了load_const处，for循环正式结束。

