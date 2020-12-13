## Python虚拟机中的函数机制

根据“一切皆对象”的原则，python中的函数也应该是一个对象。在python中，函数机制是通过python对象——PyFunctionObject来实现的。

~~~C
typedef struct {
    PyObject_HEAD
    PyObject *func_code;        /* A code object, the __code__ attribute *///对应编译后的PyCodeObject对象
    PyObject *func_globals;     /* A dictionary (other mappings won't do) *///函数运行时的globals名字空间
    PyObject *func_defaults;    /* NULL or a tuple *///默认参数
    PyObject *func_kwdefaults;  /* NULL or a dict *///关键字参数
    PyObject *func_closure;     /* NULL or a tuple of cell objects *///用于实现闭包
    PyObject *func_doc;         /* The __doc__ attribute, can be anything *///函数的文档
    PyObject *func_name;        /* The __name__ attribute, a string object *///函数__name__属性
    PyObject *func_dict;        /* The __dict__ attribute, a dict or NULL *///函数的__dict__属性
    PyObject *func_weakreflist; /* List of weak references */
    PyObject *func_module;      /* The __module__ attribute, can be anything *///函数的__module__属性
    PyObject *func_annotations; /* Annotations, a dict or NULL */
    PyObject *func_qualname;    /* The qualified name */

    /* Invariant:
     *     func_closure contains the bindings for func_code->co_freevars, so
     *     PyTuple_Size(func_closure) == PyCode_GetNumFree(func_code)
     *     (func_closure may be NULL if PyCode_GetNumFree(func_code) == 0).
     */
} PyFunctionObject;
~~~

在**Code对象与pyc文件中**我们已经知道，一段函数代码可以看作一个code block，python会将这段函数代码编译为一个code对象。code对象可以看作是一段python代码的静态表示。每一段code block只会编译产生一个code对象，这个code对象包含了源代码中的静态信息。而func对象不同，func对象是python运行时动态产生的，其中除了包含源代码中的静态信息以外（func_code指针），还包含了函数在执行时所必须的动态信息，比如，func_globals，就是函数在执行时所关联的全局作用域。

func对象和code对象的关系可以用下图来表示。

![image-20201207185721319](Python虚拟机中的函数机制.assets/image-20201207185721319.png)

### 无参函数的构造

在考察无参函数时，我们可以不去考虑python中复杂的参数传递机制。

给出一个无参函数及其编译后的python字节码

~~~python
def f():
    print("Function")
f()
#以下是编译后的字节码
  1           0 LOAD_CONST               0 (<code object f at 0x00000255F9E80190, file ".\test.py", line 1>)
              2 LOAD_CONST               1 ('f')
              4 MAKE_FUNCTION            0
              6 STORE_NAME               0 (f)

  3           8 LOAD_NAME                0 (f)
             10 CALL_FUNCTION            0
             12 POP_TOP
             14 LOAD_CONST               2 (None)
             16 RETURN_VALUE

Disassembly of <code object f at 0x00000255F9E80190, file ".\test.py", line 1>:
  2           0 LOAD_GLOBAL              0 (print)
              2 LOAD_CONST               1 ('Function')
              4 CALL_FUNCTION            1
              6 POP_TOP
              8 LOAD_CONST               0 (None)
             10 RETURN_VALUE
~~~

上述源码经过编译后产生了两个code对象，分别是源文件对应的code对象和函数f对应的code对象。我们来考察一下这两个code对象的关系。

![image-20201208190848372](Python虚拟机中的函数机制.assets/image-20201208190848372.png)

首先，我们来考察一下字节码的执行流程。两条load_const指令分别把函数f所对应的code对象和字符串“f”压入堆栈。然后执行了make_function字节码生成了函数f，随后，执行了store_name字节码将生成的函数对象以及字符串f在local命名空间中进行了关联。

我们来看一下make_function字节码的具体实现。

```C
case TARGET(MAKE_FUNCTION): {
            PyObject *qualname = POP();//一个显示从从定义该对象的模块到到达该对象(类，函数，方法)所经路径的带.号的名字
            PyObject *codeobj = POP();	//获得与函数关联的code对象
            PyFunctionObject *func = (PyFunctionObject *)
                PyFunction_NewWithQualName(codeobj, f->f_globals, qualname);//生成函数

            Py_DECREF(codeobj);
            Py_DECREF(qualname);
            if (func == NULL) {
                goto error;
            }
//////////////////////////////////////////////////////
            if (oparg & 0x08) {                     //
                assert(PyTuple_CheckExact(TOP()));	//
                func ->func_closure = POP();		//
            }										//
            if (oparg & 0x04) {						//
                assert(PyDict_CheckExact(TOP()));	//
                func->func_annotations = POP();		//
            }										///*参数处理*/
            if (oparg & 0x02) {						//
                assert(PyDict_CheckExact(TOP()));	//
                func->func_kwdefaults = POP();		//
            }										//
            if (oparg & 0x01) {						//
                assert(PyTuple_CheckExact(TOP()));	//
                func->func_defaults = POP();		//
            }										//
//////////////////////////////////////////////////////
            PUSH((PyObject *)func);
            DISPATCH();
        }
```

在make_function字节码中，最重要的是调用PyFunction_NewWithQualName函数来生成函数对象。

~~~C
PyObject *
PyFunction_NewWithQualName(PyObject *code, PyObject *globals, PyObject *qualname)
{
    PyFunctionObject *op;
    PyObject *doc, *consts, *module;
    static PyObject *__name__ = NULL;//函数的module属性

    if (__name__ == NULL) {
        __name__ = PyUnicode_InternFromString("__name__");
        if (__name__ == NULL)
            return NULL;
    }

    op = PyObject_GC_New(PyFunctionObject, &PyFunction_Type);//申请func对象空间
    if (op == NULL)
        return NULL;
//初始化对象中的各个域
    op->func_weakreflist = NULL;
    Py_INCREF(code);
    op->func_code = code;//设置code对象
    Py_INCREF(globals);
    op->func_globals = globals;//全局名字空间
    op->func_name = ((PyCodeObject *)code)->co_name;//函数名
    Py_INCREF(op->func_name);
    op->func_defaults = NULL; /* No default arguments */
    op->func_kwdefaults = NULL; /* No keyword only defaults */
    op->func_closure = NULL;//闭包
    op->vectorcall = _PyFunction_Vectorcall;

    consts = ((PyCodeObject *)code)->co_consts;
    if (PyTuple_Size(consts) >= 1) {
        doc = PyTuple_GetItem(consts, 0);
        if (!PyUnicode_Check(doc))
            doc = Py_None;
    }
    else
        doc = Py_None;
    Py_INCREF(doc);
    op->func_doc = doc;//文档

    op->func_dict = NULL;
    op->func_module = NULL;
    op->func_annotations = NULL;

    /* __module__: If module name is in globals, use it.
       Otherwise, use None. */
    module = PyDict_GetItemWithError(globals, __name__);
    if (module) {
        Py_INCREF(module);
        op->func_module = module;
    }
    else if (PyErr_Occurred()) {
        Py_DECREF(op);
        return NULL;
    }
    if (qualname)
        op->func_qualname = qualname;
    else
        op->func_qualname = op->func_name;
    Py_INCREF(op->func_qualname);

    _PyObject_GC_TRACK(op);
    return (PyObject *)op;
}
~~~

### 函数调用

在"8 LOAD_NAME 0 (f)"指令被执行之后，之前生成的函数对象f被入栈，接下来从"10 CALL_FUNCTION 0"开始，python虚拟机进入了函数调用阶段。首先看call_function指令的实现。

~~~C
case TARGET(CALL_FUNCTION): {
            PREDICTED(CALL_FUNCTION);
            PyObject **sp, *res;
            sp = stack_pointer;
            res = call_function(tstate, &sp, oparg, NULL);
            stack_pointer = sp;
            PUSH(res);
            if (res == NULL) {
                goto error;
            }
            DISPATCH();
        }
~~~

call_function指令的主要工作都是在call_function中完成的。

~~~C
Py_LOCAL_INLINE(PyObject *) _Py_HOT_FUNCTION
call_function(PyObject ***pp_stack, Py_ssize_t oparg, PyObject *kwnames)
{
    PyObject **pfunc = (*pp_stack) - oparg - 1;//获取func对象
    PyObject *func = *pfunc;//func对象
    PyObject *x, *w;
    Py_ssize_t nkwargs = (kwnames == NULL) ? 0 : PyTuple_GET_SIZE(kwnames);
    Py_ssize_t nargs = oparg - nkwargs;//参数总数
    PyObject **stack = (*pp_stack) - nargs - nkwargs;//参数出栈

    /* Always dispatch PyCFunction first, because these are
       presumed to be the most frequent callable object.
    */
    if (PyCFunction_Check(func)) {//CFunction在这里执行
        PyThreadState *tstate = _PyThreadState_GET();
        C_TRACE(x, _PyCFunction_FastCallKeywords(func, stack, nargs, kwnames));
    }
    else if (Py_TYPE(func) == &PyMethodDescr_Type) {//执行method_descriptor（比如str.upper）
        /*...*/
    }
    else {
        if (PyMethod_Check(func) && PyMethod_GET_SELF(func) != NULL) {
            /*...*///处理bound method
        }
        else {
            Py_INCREF(func);
        }

        if (PyFunction_Check(func)) {//[A]在这里开始执行用户自定义的函数
            x = _PyFunction_FastCallKeywords(func, stack, nargs, kwnames);
        }
        else {
            x = _PyObject_FastCallKeywords(func, stack, nargs, kwnames);
        }
        Py_DECREF(func);
    }

    assert((x != NULL) ^ (PyErr_Occurred() != NULL));

    /* Clear the stack of the function object. */
    while ((*pp_stack) > pfunc) {
        w = EXT_POP(*pp_stack);
        Py_DECREF(w);
    }

    return x;
}
~~~

在call_function函数中有多条分支分别处置不同类型的函数调用。用户自定义函数在标记[A]处的到了处理。

~~~C
PyObject *
_PyFunction_FastCallKeywords(PyObject *func, PyObject *const *stack,
                             Py_ssize_t nargs, PyObject *kwnames)
{
    /*必要的处理以及检查工作*/
    /* kwnames must only contains str strings, no subclass, and all keys must
       be unique */

    if (co->co_kwonlyargcount == 0 && nkwargs == 0 &&//[A]快速通道————函数声明中不包含强制关键字参数且未传入关键字参数
        (co->co_flags & ~PyCF_MASK) == (CO_OPTIMIZED | CO_NEWLOCALS | CO_NOFREE))//CO_NOFREE：没有free和cell变量
    {
        if (argdefs == NULL && co->co_argcount == nargs) {//所有参数都已经赋值：nargs——调用时传入的参数个数;co->co_argcount函数声明的参数个数（不包括强制关键字参数，*args以及**kw所声明的参数;*args所代表的参数数量也会被计算入nargs，如果有*args参数，函数流程不会进入这个分支）。
            return function_code_fastcall(co, stack, nargs, globals);
        }
        else if (nargs == 0 && argdefs != NULL
                 && co->co_argcount == PyTuple_GET_SIZE(argdefs)) {//调用时未传参数但是所有参数都具有默认值
            /* function called with no arguments, but all parameters have
               a default value: use default values as arguments .*/
            stack = _PyTuple_ITEMS(argdefs);//使用默认值作为参数值
            return function_code_fastcall(co, stack, PyTuple_GET_SIZE(argdefs),
                                          globals);
        }
    }
    //[B]不满足以上的分支条件则进入常规通道
    kwdefs = PyFunction_GET_KW_DEFAULTS(func);
    closure = PyFunction_GET_CLOSURE(func);
    name = ((PyFunctionObject *)func) -> func_name;
    qualname = ((PyFunctionObject *)func) -> func_qualname;

    if (argdefs != NULL) {
        d = _PyTuple_ITEMS(argdefs);
        nd = PyTuple_GET_SIZE(argdefs);
    }
    else {
        d = NULL;
        nd = 0;
    }
    return _PyEval_EvalCodeWithName((PyObject*)co, globals, (PyObject *)NULL,
                                    stack, nargs,
                                    nkwargs ? _PyTuple_ITEMS(kwnames) : NULL,
                                    stack + nargs,
                                    nkwargs, 1,
                                    d, (int)nd, kwdefs,
                                    closure, name, qualname);
}
~~~

可以看到，_PyFunction_FastCallKeywords的执行流程包括两个分支：标签[A]处的快速通道;标签[B]处的常规通道。快速通道与常规通道的区别在于快速通道中处理的参数类型比较简单，而常规通道对python中各种复杂的参数情况都进行了处理。

我们首先来看快速通道的后续流程。

~~~C
static PyObject* _Py_HOT_FUNCTION
function_code_fastcall(PyCodeObject *co, PyObject *const *args, Py_ssize_t nargs,
                       PyObject *globals)
{
    /*...*/
    f = _PyFrame_New_NoTrack(tstate, co, globals, NULL);//创建栈帧
    /*...*/
    result = PyEval_EvalFrameEx(f,0);//执行f

    /*...*/
    return result;
}
~~~

在function_code_fastcall中首先创建了一个frame对象，然后调用了PyEval_EvalFrameEx函数，继续对PyEval_EvalFrameEx进行追溯

```C
PyObject *
PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)
{
    PyInterpreterState *interp = _PyInterpreterState_GET_UNSAFE();
    return interp->eval_frame(f, throwflag);
}
#define _PyInterpreterState_GET_UNSAFE() (_PyThreadState_GET()->interp)
```

变量interp是当前线程状态tstate的一个成员，找到他的定义

```C
typedef struct _is {

    /*...*/
    /* Initialized to PyEval_EvalFrameDefault(). */
    _PyFrameEvalFunction eval_frame;

    /*...*/
} PyInterpreterState;
typedef PyObject* (*_PyFrameEvalFunction)(struct _frame *, int);
```

找到他的初始化函数

```C
PyInterpreterState *
PyInterpreterState_New(void)
{
    PyInterpreterState *interp = (PyInterpreterState *)
                                 PyMem_RawMalloc(sizeof(PyInterpreterState));

    if (interp == NULL) {
        return NULL;
    }

    /*...*/
    interp->eval_frame = _PyEval_EvalFrameDefault;
    /*...*/

    return interp;
}
```

最终，我们发现interp->eval_frame这个函数指针指向的就是_PyEval_EvalFrameDefault。