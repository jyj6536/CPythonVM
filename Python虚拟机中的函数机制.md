## Python虚拟机中的函数机制

根据“一切皆对象”的原则，python中的函数也应该是一个对象。在python中，函数机制是通过python对象——PyFunctionObject来实现的。

~~~C
typedef struct {
    PyObject_HEAD
    PyObject *func_code;        /* A code object, the __code__ attribute *///对应编译后的PyCodeObject对象
    PyObject *func_globals;     /* A dictionary (other mappings won't do) *///函数运行时的globals名字空间
    PyObject *func_defaults;    /* NULL or a tuple *///一个元组，储存了函数所有的默认参数
    PyObject *func_kwdefaults;  /* NULL or a dict *///保存指定了默认值的强制关键字参数的默认值
    PyObject *func_closure;     /* NULL or a tuple of cell objects *///一个元组，保存了当前函数的所有闭包层级
    PyObject *func_doc;         /* The __doc__ attribute, can be anything *///函数的文档
    PyObject *func_name;        /* The __name__ attribute, a string object *///函数__name__属性
    PyObject *func_dict;        /* The __dict__ attribute, a dict or NULL *///函数的__dict__属性
    PyObject *func_weakreflist; /* List of weak references */
    PyObject *func_module;      /* The __module__ attribute, can be anything *///函数的__module__属性，该函数所属的或者所关联的模块
    PyObject *func_annotations; /* Annotations, a dict or NULL *///对参数的注释
    PyObject *func_qualname;    /* The qualified name *///包含了该函数所属的层次结构及函数名的字符串

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

#### 快速通道

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

#### 常规通道

在常规通道中最终调用了_PyEval_EvalCodeWithName函数。\_PyEval_EvalCodeWithName函数比较复杂，我们来看一下他的部分代码。

~~~C
PyObject *
_PyEval_EvalCodeWithName(PyObject *_co, PyObject *globals, PyObject *locals,
           PyObject *const *args, Py_ssize_t argcount,
           PyObject *const *kwnames, PyObject *const *kwargs,
           Py_ssize_t kwcount, int kwstep,
           PyObject *const *defs, Py_ssize_t defcount,
           PyObject *kwdefs, PyObject *closure,
           PyObject *name, PyObject *qualname)
{
    /*...*/

    /* Create the frame */
    tstate = _PyThreadState_GET();
    assert(tstate != NULL);
    f = _PyFrame_New_NoTrack(tstate, co, globals, locals);//创建新frame对象
    /*...*/

    retval = PyEval_EvalFrameEx(f,0);//[A]执行frame对象f

	/*...*/
    return retval;
}
~~~

标签[A]处再次执行了PyEval_EvalFrameEx函数，从这里开始与快速通道一致，python虚拟机流程最终进入了_PyEval_EvalFrameDefault函数，开始了真正的“函数调用”状态。上述过程就是对x86平台函数调用过程的模拟——创建新栈帧并在新栈帧中执行代码。当流程进入虚拟机的主循环时，func对象的影响已经消失了。func对象中真正对新栈帧产生影响的是code对象和global名字空间。这也就是说，func对象的实际作用就是一个”传送工具“——对字节码指令和global名字空间进行打包传送的一种方式。

### 函数参数

pyhton中的函数支持丰富的参数类型。

+ 位置参数——最普通的参数类型，def func(a)中的a就是一个位置参数；
+ 默认参数——具有默认值的位置参数，def func(a,b=2)中的b就是一个默认参数；
+ 可变参数——可以理解为可以接收可变数量的位置参数的参数，def func(a,b,*args)中的args就是一个可变参数，当a、b分别接收对应的位置上的参数之后如果还有剩余的参数则会被args接收；
+ 关键字参数——以键值对的形式传入的参数，def func(a,b,**kw)中的kw就是一个关键字参数，kw可以接收以任意数量传入的键值对形式传入的参数，在函数中这戏参数会被组装为一个dict；
+ 强制关键字参数——关键字参数可以传入任意键名的参数，为了对键名进行限制可以采用强制关键字参数，def func(a,b,*,k1,k2=v2)或者def func(a,b,\*args,k1,k2=v2)（args是可变参数）两种方式都可以定义强制关键字参数，对强制关键字参数进行赋值时必须采用“k=v”的方式进行，强制关键字参数可以有默认值，没有默认值的强制关键字参数必须显式赋值。

#### 位置参数

##### 位置参数的传递

~~~python
##test.py
def func(a,b):
    a = a+1
    print(b)
func(1,"age")
##test.py的编译结果
  1           0 LOAD_CONST               0 (<code object func at 0x000001CAC2BB0190, file ".\test.py", line 1>)
              2 LOAD_CONST               1 ('func')
              4 MAKE_FUNCTION            0
              6 STORE_NAME               0 (func)

  4           8 LOAD_NAME                0 (func)
             10 LOAD_CONST               2 (1)
             12 LOAD_CONST               3 ('age')
             14 CALL_FUNCTION            2
             16 POP_TOP
             18 LOAD_CONST               4 (None)
             20 RETURN_VALUE

Disassembly of <code object func at 0x000001CAC2BB0190, file ".\test.py", line 1>:
  2           0 LOAD_FAST                0 (a)
              2 LOAD_CONST               1 (1)
              4 BINARY_ADD
              6 STORE_FAST               0 (a)

  3           8 LOAD_GLOBAL              0 (print)
             10 LOAD_FAST                1 (b)
             12 CALL_FUNCTION            1
             14 POP_TOP
             16 LOAD_CONST               0 (None)
             18 RETURN_VALUE
~~~

从编译后的代码可以看到，call_function指令之前有三条load指令这三条指令执行之后的运行时栈如图所示

![image-20201216200450165](Python虚拟机中的函数机制.assets/image-20201216200450165.png)

可以看到，func函数需要的参数都已经压入运行时堆栈了，接下来的call_function指令参数为2。

~~~C
Py_LOCAL_INLINE(PyObject *) _Py_HOT_FUNCTION
call_function(PyObject ***pp_stack, Py_ssize_t oparg, PyObject *kwnames)
{												//oparg = 2
    PyObject **pfunc = (*pp_stack) - oparg - 1;	//pfunc指向栈顶的func对象
    PyObject *func = *pfunc;
    PyObject *x, *w;
    Py_ssize_t nkwargs = (kwnames == NULL) ? 0 : PyTuple_GET_SIZE(kwnames);	//kwnames = 0
    Py_ssize_t nargs = oparg - nkwargs;										//nargs = 0
    PyObject **stack = (*pp_stack) - nargs - nkwargs;						//stack指向参数弹出之后的位置（1所在的位置）

  /*...*/
        if (PyFunction_Check(func)) {//执行到这里
            x = _PyFunction_FastCallKeywords(func, stack, nargs, kwnames);
        }

    /*...*/

    return x;
}
//继续追踪_PyFunction_FastCallKeywords
PyObject *					//函数对象func，栈顶指针，nargs = 2，kwnames = NULL
_PyFunction_FastCallKeywords(PyObject *func, PyObject *const *stack,
                             Py_ssize_t nargs, PyObject *kwnames)
{
    PyCodeObject *co = (PyCodeObject *)PyFunction_GET_CODE(func);
    PyObject *globals = PyFunction_GET_GLOBALS(func);
    PyObject *argdefs = PyFunction_GET_DEFAULTS(func);
    PyObject *kwdefs, *closure, *name, *qualname;
    PyObject **d;
    Py_ssize_t nkwargs = (kwnames == NULL) ? 0 : PyTuple_GET_SIZE(kwnames);
    Py_ssize_t nd;

    /*...*/
    /* kwnames must only contains str strings, no subclass, and all keys must
       be unique */

    /*...*/
        if (argdefs == NULL && co->co_argcount == nargs) {
            return function_code_fastcall(co, stack, nargs, globals);//执行到这里
        }
	/*...*/
}
//继续追溯function_code_fastcall
static PyObject* _Py_HOT_FUNCTION        //args的值即是之前的stack参数的值
function_code_fastcall(PyCodeObject *co, PyObject *const *args, Py_ssize_t nargs,
                       PyObject *globals)
{
    PyFrameObject *f;
    PyThreadState *tstate = _PyThreadState_GET();
    PyObject **fastlocals;
    Py_ssize_t i;
    PyObject *result;
	/*...*/
    f = _PyFrame_New_NoTrack(tstate, co, globals, NULL);
    if (f == NULL) {
        return NULL;
    }

    fastlocals = f->f_localsplus;

    for (i = 0; i < nargs; i++) {//[A]
        Py_INCREF(*args);
        fastlocals[i] = *args++;
    }
    result = PyEval_EvalFrameEx(f,0);
    /*...*/
}
~~~

随着对python虚拟机执行过程的一步步追溯，我们在function_code_fastcall中发现了熟悉的frame对象以及f_localsplus这个frame的成员。参数args的值就是之前call_function中stack的值。

![image-20201216203307113](Python虚拟机中的函数机制.assets/image-20201216203307113.png)

标签[A]处的for对新建栈帧的f_localsplus进行了循环赋值的操作，f->f_localsplus[0]、f->f_localsplus[1]分别被赋值为对象1、对象"age"。根据之前《Python虚拟机框架》中对frame对象结构的描述可知，f_localsplus是由两部分空间组成的——locals变量空间和运行时栈空间，函数参数也是一种局部变量，所以在这里func的参数a、b被存储到f_localsplus的起始位置与我们之前对frame对象的考察时完全符合的。

##### 位置参数的访问

当函数参数存储完成之后就进入了执行阶段。首先映入眼帘的就是一条load_fast指令。

~~~C
case TARGET(LOAD_FAST): {
            PyObject *value = GETLOCAL(oparg);
            if (value == NULL) {
                format_exc_check_arg(PyExc_UnboundLocalError,
                                     UNBOUNDLOCAL_ERROR_MSG,
                                     PyTuple_GetItem(co->co_varnames, oparg));
                goto error;
            }
            Py_INCREF(value);
            PUSH(value);//获取fastlocals
            FAST_DISPATCH();
        }
#define GETLOCAL(i)     (fastlocals[i])
~~~

load_fast指令是以f_localsplus这片内存为操作对象的指令。load_fast 0这条指令的作用就是将f_localsplus下标为0处的对象压入运行时堆栈，store_fast也类似，只不过是操作方向相反。

~~~c
case TARGET(STORE_FAST): {
            PREDICTED(STORE_FAST);
            PyObject *value = POP();
            SETLOCAL(oparg, value);//设置fastlocals
            FAST_DISPATCH();
        }
#define SETLOCAL(i, value)      do { PyObject *tmp = GETLOCAL(i); \
                                     GETLOCAL(i) = value; \
                                     Py_XDECREF(tmp); } while (0)
~~~

到这里，我们对pyhton中是如何传递位置参数，以及在函数调用过程中是如何访问位置参数，都已经有了比较清晰的了解。在调用函数时，python虚拟机将函数参数从左到右压入运行时堆栈，在function_code_fastcall中，虚拟机又将这些参数依次拷贝到与函数对应的frame对象的f_localsplus域中。在访问这些参数时，python虚拟机并没有去查找名字空间，而是通过load_fast、store_fast指令借助偏移量来访问f_localsplus所存储的对象。这种通过索引来访问参数的方法正是“位置参数”名称的由来。

#### 默认参数

##### 默认参数的储存

在python中，允许位置参数具有默认值。具有默认值的位置参数与普通的位置参数具有不同的实现机制。在这里，我们深入考察具有默认值的位置参数的实现机制。

~~~python
def func(a=1,b=2):
    print(a+b)
func()
func(b=3)
#test.py的编译结果
  1           0 LOAD_CONST               7 ((1, 2))
              2 LOAD_CONST               2 (<code object func at 0x00000201728BFDF0, file ".\test.py", line 1>)
              4 LOAD_CONST               3 ('func')
              6 MAKE_FUNCTION            1 (defaults)
              8 STORE_NAME               0 (func)

  3          10 LOAD_NAME                0 (func)
             12 CALL_FUNCTION            0
             14 POP_TOP

  4          16 LOAD_NAME                0 (func)
             18 LOAD_CONST               4 (3)
             20 LOAD_CONST               5 (('b',))
             22 CALL_FUNCTION_KW         1
             24 POP_TOP
             26 LOAD_CONST               6 (None)
             28 RETURN_VALUE

Disassembly of <code object func at 0x00000201728BFDF0, file ".\test.py", line 1>:
  2           0 LOAD_GLOBAL              0 (print)
              2 LOAD_FAST                0 (a)
              4 LOAD_FAST                1 (b)
              6 BINARY_ADD
              8 CALL_FUNCTION            1
             10 POP_TOP
             12 LOAD_CONST               0 (None)
             14 RETURN_VALUE
~~~

通过观察func所对应的字节码我们可以发现，虽然参数a、b具有默认值，但是函数中对a、b的访问仍然是通过位置参数访问指令实现的。也就是说，是否具有默认值不会对位置参数在函数内部的访问方式产生影响。那么，默认参数的玄机到底体现在哪里？通过观察make_function函数的参数，我们可以从中发现一些端倪。

在参数具有默认值的情况下，make_function的参数为1，而没有默认值时无论有没有参数make_function的参数都是0。再次查看make_function字节码的部分实现。

~~~C
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
            if (oparg & 0x01) {						///*oparg为1，执行该分支*/
                assert(PyTuple_CheckExact(TOP()));	//
                func->func_defaults = POP();		///[A]
            }										//
//////////////////////////////////////////////////////
~~~

在执行字节码make_function之前，虚拟机执行了三条load_const分别将一个tuple对象((1,2)，存储参数a与b的默认值)，func所对应的code对象以及unicode对象（func的qualname）压入堆栈，当执行到标签[A]处时，code对象与unicode对象已经出栈，此时出栈的就是最先入栈的tuple对象，所以最终返回的func对象func_defaults指向的就是这个tuple对象。通过这样一些列的操作，func的默认值与func对象成功绑定到了一起。

##### func的第一次调用

首先，我们来看函数func的第一次访问call_function 0。

重新追溯字节码call_function的执行过程，我们发现了针对不传参数但是所有参数都具有默认值的情况的处理。

~~~C
PyObject *
_PyFunction_FastCallKeywords(PyObject *func, PyObject *const *stack,
                             Py_ssize_t nargs, PyObject *kwnames)
{
    PyObject *argdefs = PyFunction_GET_DEFAULTS(func);
    /*必要的处理以及检查工作*/
    /* kwnames must only contains str strings, no subclass, and all keys must
       be unique */

   /*...*/
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
    /*...*/
}
~~~

查看_PyTuple_ITEMS的定义

~~~C
#define _PyTuple_ITEMS(op) (_PyTuple_CAST(op)->ob_item)
~~~

在这里，argdefs指向的时func对象的func_defaults域，也就是存储func对象默认参数值的tuple。结合tuple对象的内存布局我们可以知道，此处的stack指针指向的就是tuple中存储元素的内存区域。这就意味着tuple中的元素被作为参数传递给了function_code_fastcall。之后虚拟机中的执行流程就与普通位置参数的执行流程一致了。

##### func的第二次调用

首先，我们来看一下执行func之前的运行时堆栈情况。

![image-20201221174058157](Python虚拟机中的函数机制.assets/image-20201221174058157.png)

在func的第二次调用中，我们给b赋了一个指定值3。从编译得到的字节码中可以发现，此时对func的调用已经不是通过call_function，而是通过call_function_kw。

~~~C
case TARGET(CALL_FUNCTION_KW): {
            PyObject **sp, *res, *names;

            names = POP();
            assert(PyTuple_CheckExact(names) && PyTuple_GET_SIZE(names) <= oparg);
            sp = stack_pointer;
            res = call_function(&sp, oparg, names);
            stack_pointer = sp;
            PUSH(res);
            Py_DECREF(names);

            if (res == NULL) {
                goto error;
            }
            DISPATCH();
        }
~~~

与字节码call_function相比，call_fucntion_kw中调用call_function函数时所传的第三个参数不再是NULL，而是栈顶的tuple对象。同时，oparg的值为1。我们带着这些信息重新考察call_function函数。

~~~C
Py_LOCAL_INLINE(PyObject *) _Py_HOT_FUNCTION
call_function(PyObject ***pp_stack, Py_ssize_t oparg, PyObject *kwnames)
{
    PyObject **pfunc = (*pp_stack) - oparg - 1;
    PyObject *func = *pfunc;
    PyObject *x, *w;
    Py_ssize_t nkwargs = (kwnames == NULL) ? 0 : PyTuple_GET_SIZE(kwnames);
    Py_ssize_t nargs = oparg - nkwargs;
    PyObject **stack = (*pp_stack) - nargs - nkwargs;
	/*...*/
~~~

![image-20201221180955838](Python虚拟机中的函数机制.assets/image-20201221180955838.png)

初始化后的各个变量的值如上图所示。

继续进行追溯，找到\_PyEval_EvalCodeWithName函数。\_PyEval_EvalCodeWithNam函数是一个综合应对python中不同情况参数的函数，在这里我们将与本节讨论相关的部分截取出来。

~~~C
PyObject *
_PyEval_EvalCodeWithName(PyObject *_co, PyObject *globals, PyObject *locals,//func对象对应的code对象，func对象的全局名字空间，func对象的本地名字空间
           PyObject *const *args, Py_ssize_t argcount,//上图中的stack，上图中的nargs（0）
           PyObject *const *kwnames, PyObject *const *kwargs,//上图中的names（一个tuple对象）的元素域，上图中的stack
           Py_ssize_t kwcount, int kwstep,//nkwargs，1
           PyObject *const *defs, Py_ssize_t defcount,//func的func_defaults域(一个tuple对象)的元素域，元素域的元素数
           PyObject *kwdefs, PyObject *closure,//func的func_kwdefaults域（储存了具有默认值的强制关键字参数的默认值），闭包
           PyObject *name, PyObject *qualname)//函数的name以及qualname
{
    /*...*/
    const Py_ssize_t total_args = co->co_argcount + co->co_kwonlyargcount;
    
    /* Copy positional arguments into local variables *///[A]
    if (argcount > co->co_argcount) {//在这里，argcount代表函数调用时按位置传入的位置函数参数个数，co_argcount代表函数声明中的位置参数个数；本例中argcount为0是因为调用时未采用按位置传入的形式传入位置参数，如果调用形式为func(2,b=3)，那么此处的argcount为1，同时args[0]存放long对象2，后续的for循环就会调用SETLOCAL宏对frame对象的fastlocals[0]进行赋值从而实现局部变量a的初始化
        n = co->co_argcount;
    }
    else {
        n = argcount;
    }
    for (i = 0; i < n; i++) {
        x = args[i];
        Py_INCREF(x);
        SETLOCAL(i, x);
    }
    
    /*...*/
    
    /* Handle keyword arguments passed as two strided arrays *///[B]
    kwcount *= kwstep;//从这里开始，同时处理了关键字参数以及默认参数的情况
    for (i = 0; i < kwcount; i += kwstep) {
        PyObject **co_varnames;//('a','b')
        PyObject *keyword = kwnames[i];	//kwnames:['b']
        PyObject *value = kwargs[i];	//kwargs：栈空间中存储给位置参数所赋的值的起始地址，这里指向3
        Py_ssize_t j;					//变量total_args代表的是位置参数以及强制关键字参数的个数总和

        if (keyword == NULL || !PyUnicode_Check(keyword)) {
            PyErr_Format(PyExc_TypeError,
                         "%U() keywords must be strings",
                         co->co_name);
            goto fail;
        }

        /* Speed hack: do raw pointer compares. As names are
           normally interned this should almost always hit. */
        co_varnames = ((PyTupleObject *)(co->co_varnames))->ob_item;
        for (j = 0; j < total_args; j++) {//在co_varnames中，位置参数以及强制关键字参数排在前面，以def func(a=1,b=2,*args,k,m,**kw)为例，该函数对象的对应的code对象的co_varnames为('a', 'b', 'k', 'm', 'args', 'kw')。所以python虚拟机在这里只需要循环co_varnames的前total_args个对象；下标j对应的就是该参数在fastlocals中所对应的下标
            PyObject *name = co_varnames[j];
            if (name == keyword) {//根据参数名确定是否找到被赋值的参数
                goto kw_found;
            }
        }

        /* Slow fallback, just in case */
        for (j = 0; j < total_args; j++) {
            PyObject *name = co_varnames[j];
            int cmp = PyObject_RichCompareBool( keyword, name, Py_EQ);
            if (cmp > 0) {
                goto kw_found;
            }
            else if (cmp < 0) {
                goto fail;
            }
        }

        assert(j >= total_args);
        if (kwdict == NULL) {//kwdict是用于接收**kw类型参数的字典
            PyErr_Format(PyExc_TypeError,
                         "%U() got an unexpected keyword argument '%S'",
                         co->co_name, keyword);
            goto fail;
        }

        if (PyDict_SetItem(kwdict, keyword, value) == -1) {
            goto fail;
        }
        continue;

      kw_found://根据下标j为对应的参数进行赋值，value是在堆栈中的参数值
        if (GETLOCAL(j) != NULL) {
            PyErr_Format(PyExc_TypeError,
                         "%U() got multiple values for argument '%S'",
                         co->co_name, keyword);
            goto fail;
        }
        Py_INCREF(value);
        SETLOCAL(j, value);
    }
    /*...*/
     /* Add missing positional arguments (copy default values from defs) *///[C]
    if (argcount < co->co_argcount) {//为没有赋值的位置参数赋默认值
        Py_ssize_t m = co->co_argcount - defcount;//m是没有默认值的默认参数的个数
        Py_ssize_t missing = 0;
        for (i = argcount; i < m; i++) {//argcount是函数调用时传入的按位置赋值的参数个数；在co_varnames中，没有默认值的位置参数排在默认参数（有默认值的位置参数）之前，而参数在co_varnames中顺序与其在frame对象的fastlocals域中的顺序是对应的，所以在这里，函数的第argcount到第m个位置参数都应该已经被赋值（通过key=value的赋值方式），否则说明位置参数缺失
            if (GETLOCAL(i) == NULL) {
                missing++;
            }
        }
        if (missing) {
            missing_arguments(co, missing, defcount, fastlocals);
            goto fail;
        }
        if (n > m)//执行到这里说明第m个参数之前的参数都已经处理无误；如果n>m，那么func_defaults的前n-m个值所对应的位置参数肯定都已经被赋值
            i = n - m;
        else
            i = 0;//n<=m，说明所有默认参数都有可能没被赋值
        for (; i < defcount; i++) {//函数的第m个参数开始都是有默认值的（存储在函数的func_defaults域中，值的存储顺序与默认参数的声明顺序相同）。
            if (GETLOCAL(m+i) == NULL) {
                PyObject *def = defs[i];
                Py_INCREF(def);
                SETLOCAL(m+i, def);
            }
        }
    }
    /*...*/
}
~~~

通过对源码的分析，我们对默认参数的赋值过程已经有了大体上的了解。接下来我们通过一个例子对位置参数的赋值过程进行总结。

假设现在有一个函数f的定义为

~~~Python
def f(a,b,c=1,d=2)
~~~

调用代码为

~~~python
f(1,b=1,c=3)
~~~

我们画出f被调用时的运行时堆栈，f对应的frame对象的fastlocals域，f对应的code对象的co_varnames域以及f的func_defaults域。

![image-20201225170228435](Python虚拟机中的函数机制.assets/image-20201225170228435.png)

根据对源码的分析可知，python虚拟机对位置参数的赋值是按照有赋值的无默认值的位置参数、有赋值的默认参数、未赋值的默认参数的顺序进行的。这些赋值动作分别在源代码[A]、[B]、[C]处进行。

![image-20201225202107239](Python虚拟机中的函数机制.assets/image-20201225202107239.png)

#### 可变参数

pyhton中的函数支持可变数量的位置参数的传递——通过*args的方式来接受多余的位置参数。

我们通过一段代码来考察python中的可变参数是如何传递的。

~~~Python
def func(a,b,*args):
    pass
func(1,2,3,4,5,6)
##编译后的python字节码
 1           0 LOAD_CONST               0 (<code object func at 0x000001E4BF6A0DF0, file ".\test.py", line 1>)
              2 LOAD_CONST               1 ('func')
              4 MAKE_FUNCTION            0
              6 STORE_NAME               0 (func)

  3           8 LOAD_NAME                0 (func)
             10 LOAD_CONST               2 (1)
             12 LOAD_CONST               3 (2)
             14 LOAD_CONST               4 (3)
             16 LOAD_CONST               5 (4)
             18 LOAD_CONST               6 (5)
             20 LOAD_CONST               7 (6)
             22 CALL_FUNCTION            6
             24 POP_TOP
             26 LOAD_CONST               8 (None)
             28 RETURN_VALUE

Disassembly of <code object func at 0x000001E4BF6A0DF0, file ".\test.py", line 1>:
  2           0 LOAD_CONST               0 (None)
              2 RETURN_VALUE
~~~

画出函数调用时主调函数的运行时堆栈以及func对象所对应的code对象的co_varnames域、frame对象的fastlocals域。

![image-20201229191027373](Python虚拟机中的函数机制.assets/image-20201229191027373.png)

追溯虚拟机的执行流程，再次考察_PyEval_EvalCodeWithName函数

~~~C
PyObject *
_PyEval_EvalCodeWithName(PyObject *_co, PyObject *globals, PyObject *locals,
           PyObject *const *args, Py_ssize_t argcount,
           PyObject *const *kwnames, PyObject *const *kwargs,
           Py_ssize_t kwcount, int kwstep,
           PyObject *const *defs, Py_ssize_t defcount,
           PyObject *kwdefs, PyObject *closure,
           PyObject *name, PyObject *qualname)
{
    /*...*/
    /* Copy positional arguments into local variables */
    if (argcount > co->co_argcount) {//[A]
        n = co->co_argcount;
    }
    else {
        n = argcount;
    }
    for (i = 0; i < n; i++) {
        x = args[i];
        Py_INCREF(x);
        SETLOCAL(i, x);
    }

    /* Pack other positional arguments into the *args argument */
    if (co->co_flags & CO_VARARGS) {//CO_VARARGS位被设置说明函数参数中声明了可变参数
        u = PyTuple_New(argcount - n);//根据标签[A]处对n的处理可知，n一定是小于等于argcount，所以u是大于等于零的
        if (u == NULL) {
            goto fail;
        }
        SETLOCAL(total_args, u);//使用tuple来接收可变位置参数
        for (i = n; i < argcount; i++) {//在argcount > co->co_argcount时，说明函数实际接收的参数中是有多余的位置参数的，需要用可变参数来接收
            x = args[i];
            Py_INCREF(x);
            PyTuple_SET_ITEM(u, i-n, x);
        }
    }
    /*...*/
}
~~~

python虚拟机接收可变参数的过程已经清晰地展现在我们面前了——在位置参数赋值完成之后，如果还有剩余的按位置传入的参数，那么就新建一个一个tuple来接收这些参数。

![image-20201229191532600](Python虚拟机中的函数机制.assets/image-20201229191532600.png)

#### 关键字参数以及强制关键字参数

python中强制关键字参数是必须以key-value形式赋值的参数。关键字参数是以**kw形式声明的可以接收任意数量key-value形式参数的参数。

~~~Python
def func(a,b,*,k=1,d,w=3,**kw):
    pass
func(1,2,d=2,w=4,i=10,j=20,z=30)
##编译后的python字节码
  1           0 LOAD_CONST               0 (1)
              2 LOAD_CONST               1 (3)
              4 LOAD_CONST               2 (('k', 'w'))
              6 BUILD_CONST_KEY_MAP      2
              8 LOAD_CONST               3 (<code object func at 0x00000288C4D20DF0, file ".\test.py", line 1>)
             10 LOAD_CONST               4 ('func')
             12 MAKE_FUNCTION            2 (kwdefaults)
             14 STORE_NAME               0 (func)

  3          16 LOAD_NAME                0 (func)
             18 LOAD_CONST               0 (1)
             20 LOAD_CONST               5 (2)
             22 LOAD_CONST               5 (2)
             24 LOAD_CONST               6 (4)
             26 LOAD_CONST               7 (10)
             28 LOAD_CONST               8 (20)
             30 LOAD_CONST               9 (30)
             32 LOAD_CONST              10 (('d', 'w', 'i', 'j', 'z'))
             34 CALL_FUNCTION_KW         7
             36 POP_TOP
             38 LOAD_CONST              11 (None)
             40 RETURN_VALUE

Disassembly of <code object func at 0x00000288C4D20DF0, file ".\test.py", line 1>:
  2           0 LOAD_CONST               0 (None)
              2 RETURN_VALUE
~~~

首先，来看一下字节码BUILD_CONST_KEY_MAP。

~~~C
case TARGET(BUILD_CONST_KEY_MAP): {
            Py_ssize_t i;
            PyObject *map;
            PyObject *keys = TOP();
            if (!PyTuple_CheckExact(keys) ||
                PyTuple_GET_SIZE(keys) != (Py_ssize_t)oparg) {
                PyErr_SetString(PyExc_SystemError,
                                "bad BUILD_CONST_KEY_MAP keys argument");
                goto error;
            }
            map = _PyDict_NewPresized((Py_ssize_t)oparg);
            if (map == NULL) {
                goto error;
            }
            for (i = oparg; i > 0; i--) {
                int err;
                PyObject *key = PyTuple_GET_ITEM(keys, oparg - i);
                PyObject *value = PEEK(i + 1);
                err = PyDict_SetItem(map, key, value);
                if (err != 0) {
                    Py_DECREF(map);
                    goto error;
                }
            }

            Py_DECREF(POP());
            while (oparg--) {
                Py_DECREF(POP());
            }
            PUSH(map);
            DISPATCH();
        }
~~~

BUILD_CONST_KEY_MAP的作用是构造一个dict对象并压入运行时栈的栈顶。

![image-20201231192835830](Python虚拟机中的函数机制.assets/image-20201231192835830.png)

在BUILD_CONST_KEY_MAP执行之前，虚拟机首先把key-value对的value依次压入堆栈，然后将所有的key组合成为一个tuple并压入堆栈。在构造dict时，虚拟机将堆栈中的value与tuple中的key按顺序对应，调用PyDict_SetItem函数将key-value对置入dict对象中。构造结束后，将dict对象压入运行时堆栈。

了解了BUILD_CONST_KEY_MAP指令之后，我们可以画出堆栈make_function指令执行之前的运行时堆栈。

![image-20201231194149195](Python虚拟机中的函数机制.assets/image-20201231194149195.png)

字节码make_fucntion的参数是2，重新考察make_function的实现

~~~C
if (oparg & 0x02) {
    assert(PyDict_CheckExact(TOP()));
    func->func_kwdefaults = POP();
}	
~~~

可以发现，强制关键字参数的默认值被存储到了func对象的func_kwdefaults域中。

<img src="Python虚拟机中的函数机制.assets/image-20201231200840727.png" alt="image-20201231200840727" style="zoom:67%;" />

从运行时堆栈的布局我们可以得出结论，无论是默认参数还是关键字参数，只要是以key-value形式赋值的参数，虚拟机都是先将value按照顺序压入运行时堆栈，然后再将key按顺序组合成一个tuple并压入堆栈。



~~~C
PyObject *
_PyEval_EvalCodeWithName(...)
{
    /*...*/
    
    /* Handle keyword arguments passed as two strided arrays */
    kwcount *= kwstep;//从这里开始，同时处理了关键字参数以及默认参数的情况；kwcount是以key-value形式传入的参数的个数
    for (i = 0; i < kwcount; i += kwstep) {
        PyObject **co_varnames;//cdoe对象的co_varnames域，('a','b','k','d','w','kw')
        PyObject *keyword = kwnames[i];	//kwnames——('d', 'w', 'i', 'j', 'z')
        PyObject *value = kwargs[i];	//传入的key-value参数：kwnames[i]-knames[i]
        Py_ssize_t j;					//变量total_args代表的是位置参数以及强制关键字参数的个数总和

        if (keyword == NULL || !PyUnicode_Check(keyword)) {
            PyErr_Format(PyExc_TypeError,
                         "%U() keywords must be strings",
                         co->co_name);
            goto fail;
        }

        /* Speed hack: do raw pointer compares. As names are
           normally interned this should almost always hit. */
        co_varnames = ((PyTupleObject *)(co->co_varnames))->ob_item;
        for (j = 0; j < total_args; j++) {//在co_varnames中，位置参数以及强制关键字参数排在前面，以def func(a=1,b=2,*args,k,m,**kw)为例，该函数对象的对应的code对象的co_varnames为('a', 'b', 'k', 'm', 'args', 'kw')。所以python虚拟机在这里只需要循环co_varnames的前total_args个对象；下标j对应的就是该参数在fastlocals中所对应的下标
            PyObject *name = co_varnames[j];
            if (name == keyword) {//根据参数名确定是否找到被赋值的参数
                goto kw_found;
            }
        }

        /* Slow fallback, just in case */
        for (j = 0; j < total_args; j++) {
            PyObject *name = co_varnames[j];
            int cmp = PyObject_RichCompareBool( keyword, name, Py_EQ);
            if (cmp > 0) {
                goto kw_found;
            }
            else if (cmp < 0) {
                goto fail;
            }
        }

        assert(j >= total_args);//如果key-value参数的key不在co_varnames中，则说明这个key应该出现在关键字参数中（被**kw接收）
        if (kwdict == NULL) {//kwdict是用于接收**kw类型参数的字典
            PyErr_Format(PyExc_TypeError,
                         "%U() got an unexpected keyword argument '%S'",
                         co->co_name, keyword);
            goto fail;
        }

        if (PyDict_SetItem(kwdict, keyword, value) == -1) {
            goto fail;
        }
        continue;

      kw_found://根据下标j为对应的参数进行赋值，value是在堆栈中的参数值
        if (GETLOCAL(j) != NULL) {
            PyErr_Format(PyExc_TypeError,
                         "%U() got multiple values for argument '%S'",
                         co->co_name, keyword);
            goto fail;
        }
        Py_INCREF(value);
        SETLOCAL(j, value);
    }
    /*...*/
      /* Add missing keyword arguments (copy default values from kwdefs) */
    if (co->co_kwonlyargcount > 0) {//在这里，对强制关键字参数进行了检查——对于依然没有赋值的强制关键字参数，检查func对象的func_kwdefaults中是否存有该关键字参数的默认值，如果没有，那么说明传参时缺少参数，需要抛出异常。
        Py_ssize_t missing = 0;
        for (i = co->co_argcount; i < total_args; i++) {
            PyObject *name;
            if (GETLOCAL(i) != NULL)
                continue;
            name = PyTuple_GET_ITEM(co->co_varnames, i);
            if (kwdefs != NULL) {
                PyObject *def = PyDict_GetItem(kwdefs, name);
                if (def) {
                    Py_INCREF(def);
                    SETLOCAL(i, def);
                    continue;
                }
            }
            missing++;
        }
        if (missing) {
            missing_arguments(co, missing, -1, fastlocals);
            goto fail;
        }
    }
    /*...*/
}
~~~

最终的frame对象中的fastlocals域如下图所示。

![image-20201231204445740](Python虚拟机中的函数机制.assets/image-20201231204445740.png)

#### 闭包

在python中，闭包的创建通常都是利用嵌套函数来完成的。在code对象、frame对象以及func对象中分别有与闭包的实现相关的域。

code对象中与闭包相关的域是co_cellvars和co_freevars。两者的具体含义如下：

+ co_cellvars：通常是一个tuple，保存嵌套的作用域中使用的变量名集合。
+ co_freevars：通常是一个tuple，保存了本作用域中使用的外层作用域中的变量名集合。

考虑如下代码

~~~python
def get_func():
    value = "outer"
    def inner_func():
        return value
    return inner_func
print("cellvars of get_func",get_func.__code__.co_cellvars)
print("freevars of get_func",get_func.__code__.co_freevars)
ineer = get_func()
print("cellvar of ineer",ineer.__code__.co_cellvars)
print("freevars of ineer",ineer.__code__.co_freevars)
##输出
cellvars of get_func ('value',)
freevars of get_func ()
cellvar of ineer ()
freevars of ineer ('value',)
~~~

在frame对象中也有域闭包相关的域——f_localsplus。

~~~C
extras = code->co_nlocals + ncells + nfrees;
~~~

![image-20210104194133325](Python虚拟机中的函数机制.assets/image-20210104194133325.png)

extras是图中黄色区域的内存大小。从源代码可以看出，这部分内存由三部分组成——局部变量，cell变量以及free变量。

在func对象中，也有与闭包实现相关的域。

~~~python
def get_func(a,b=1):
    value = "outer"
    b += 1
    c = 1
    c += 1
    def inner_func():
        print(a)
        print(b)
        print(value)
    return inner_func
##编译后的python字节码
  1           0 LOAD_CONST               4 ((1,))
              2 LOAD_CONST               1 (<code object get_func at 0x000001E28E741EA0, file ".\test.py", line 1>)
              4 LOAD_CONST               2 ('get_func')
              6 MAKE_FUNCTION            1 (defaults)
              8 STORE_NAME               0 (get_func)

 11          10 LOAD_CONST               3 (None)
             12 RETURN_VALUE

Disassembly of <code object get_func at 0x000001E28E741EA0, file ".\test.py", line 1>:
  2           0 LOAD_CONST               1 ('outer')
              2 STORE_DEREF              2 (value)

  3           4 LOAD_DEREF               1 (b)
              6 LOAD_CONST               2 (1)
              8 INPLACE_ADD
             10 STORE_DEREF              1 (b)

  4          12 LOAD_CONST               2 (1)
             14 STORE_FAST               2 (c)

  5          16 LOAD_FAST                2 (c)
             18 LOAD_CONST               2 (1)
             20 INPLACE_ADD
             22 STORE_FAST               2 (c)

  6          24 LOAD_CLOSURE             0 (a)
             26 LOAD_CLOSURE             1 (b)
             28 LOAD_CLOSURE             2 (value)
             30 BUILD_TUPLE              3
             32 LOAD_CONST               3 (<code object inner_func at 0x000001E28E741DF0, file ".\test.py", line 6>)
             34 LOAD_CONST               4 ('get_func.<locals>.inner_func')
             36 MAKE_FUNCTION            8 (closure)
             38 STORE_FAST               3 (inner_func)

 10          40 LOAD_FAST                3 (inner_func)
             42 RETURN_VALUE

Disassembly of <code object inner_func at 0x000001E28E741DF0, file ".\test.py", line 6>:
  7           0 LOAD_GLOBAL              0 (print)
              2 LOAD_DEREF               0 (a)
              4 CALL_FUNCTION            1
              6 POP_TOP

  8           8 LOAD_GLOBAL              0 (print)
             10 LOAD_DEREF               1 (b)
             12 CALL_FUNCTION            1
             14 POP_TOP

  9          16 LOAD_GLOBAL              0 (print)
             18 LOAD_DEREF               2 (value)
             20 CALL_FUNCTION            1
             22 POP_TOP
             24 LOAD_CONST               0 (None)
             26 RETURN_VALUE
~~~

在编译得到的字节码中，我们发现了许多新的字节码。顺着虚拟机的执行流程，我们逐一对这些字节码进行分析。

首先，从对frame对象的分析可以得知，frame对象是运行时动态生成的，而code对象中的闭包信息是编译时就已经静态确定的，所以在分析get_func以及inner_func中的字节码之前，我们还是要回到call_function指令中对虚拟机是如何将code对象中的静态闭包信息转换为frame对象中的动态闭包信息进行分析。

观察get_func函数编译后的字节码可以得出以下结论：

在get_func中，如果操作的局部变量是cell变量（被内层函数引用的变量，比如a、b以及value），使用的是字节码store_deref与load_deref，这与普通局部变量（比如c）使用的字节码load_fast以及store_fast是不同的。粗略观察发现字节码store_deref与load_deref操作的是f_localsplus域中的cell变量区，这说明在func_get函数中的字节码执行之前，cell变量区已经初始化完毕，然而func_get所对应的字节码中并没有与cell变量区初始化相关的字节码。所以，cell变量区的初始化一定是在get_func函数中的字节码执行之前就已经完成了（_PyEval_EvalCodeWithName中）。

将_PyEval_EvalCodeWithName中处理cell变量区的相关部分截取出来

~~~C
/* Allocate and initialize storage for cell vars, and copy free
       vars into frame. */
    for (i = 0; i < PyTuple_GET_SIZE(co->co_cellvars); ++i) {//在这个for循环中对get_func函数所对应的frame对象的f_localsplus域中的cell变量区进行了初始化。
        PyObject *c;
        Py_ssize_t arg;
        /* Possibly account for the cell variable being an argument. */
        if (co->co_cell2arg != NULL &&
            (arg = co->co_cell2arg[i]) != CO_CELL_NOT_AN_ARG) {//在这里需要考虑cell变量是一个参数变量的情况。首先要明确一点，当_PyEval_EvalCodeWithName的流程执行到这里时，所有的参数都已经赋值完毕。co->co_cell2arg[i]指示第i个cell变量是否是一个参数变量。如果是，那么co->co_cell2arg[i]的值arg就是该变量在f_localsplus域中的局部变量域的索引值。虚拟机使用该值创建一个新的PyCell对象。
            c = PyCell_New(GETLOCAL(arg));
            /* Clear the local copy. */
            SETLOCAL(arg, NULL);
        }
        else {
            c = PyCell_New(NULL);//流程执行到这里说明第i个cell变量是一个普通的局部变量（非参数），初始化一个空的PyCell对象。之所以初始化为空的PyCell对象，是因为对该cell变量的赋值操作（字节码store_deref）还没有执行。
        }
        if (c == NULL)
            goto fail;
        SETLOCAL(co->co_nlocals + i, c);
    }
~~~

在上面的代码中，我们遇到了一个新的python对象——PyCell对象。

~~~C
typedef struct {
    PyObject_HEAD
    PyObject *ob_ref;       /* Content of the cell or NULL when empty */
} PyCellObject;
~~~

这个对象非常简单，仅仅维护了一个ob_ref指针，指向python中的对象。ob_ref指针在PyCell_New函数中被设置。