# Python运行环境初始化

python启动后，有意义的初始化动作是从Py_Initialize开始的。在Py_Initialize，仅有一个函数被调用，即Py_InitializeEx。

~~~C
void
Py_Initialize(void)
{
    Py_InitializeEx(1);//调用Py_InitializeEx
}

void
Py_InitializeEx(int install_sigs)
{
    if (_PyRuntime.initialized) {//Py_Initialize已经被调用过了
        /* bpo-33932: Calling Py_Initialize() twice does nothing. */
        return;
    }

    _PyInitError err;
    _PyCoreConfig config = _PyCoreConfig_INIT;//根据环境变量以及命令行参数设置该结构体影响python虚拟机的运行时行为
    config.install_signal_handlers = install_sigs;

    err = _Py_InitializeFromConfig(&config);//执行核心初始化动作
    _PyCoreConfig_Clear(&config);

    if (_Py_INIT_FAILED(err)) {
        _Py_FatalInitError(err);
    }
}
~~~

在Py_InitializeEx中，主要的初始化动作都是在_Py_InitializeFromConfig中完成的，其中，主要的工作又是在\_Py_InitializeCore中进行的，在这里，虚拟机进行了最重要的进程以及线程的初始化过程。

在《Python虚拟机框架》中，我们已经讨论过Pyhton的运行模型。python虚拟机分别使用了**PyInterpreterState**以及**PyThreadState**这两个结构体对进程以及线程进行了模拟。在\_Py_InitializeCore中，虚拟机除了进行必要的配置工作以外，最终又调用了\_Py_InitializeCore_impl进行了虚拟机中第一个is和ts对象的初始化。

~~~C
static _PyInitError
_Py_InitializeCore_impl(PyInterpreterState **interp_p,
                        const _PyCoreConfig *core_config)
{
    PyInterpreterState *interp;

    _PyInitError err = pycore_init_runtime(core_config);//_PyRuntime是一个_PyRuntimeState类型的全局对象，用来保存完整的运行时状态
    if (_Py_INIT_FAILED(err)) {
        return err;
    }

    err = pycore_create_interpreter(core_config, &interp);//初始化第一个is和ts对象
    /***/
}

static _PyInitError
pycore_create_interpreter(const _PyCoreConfig *core_config,
                          PyInterpreterState **interp_p)
    PyInterpreterState *interp = PyInterpreterState_New();
/*...*/
PyThreadState *tstate = PyThreadState_New(interp);
/*...*/
}
~~~

is对象的创建

~~~C
PyInterpreterState *
PyInterpreterState_New(void)
{
    PyInterpreterState *interp = (PyInterpreterState *)
                                 PyMem_RawMalloc(sizeof(PyInterpreterState));

    if (interp == NULL) {
        return NULL;
    }

    //设置is对象的各个域
    /*...*/

    HEAD_LOCK();
    if (_PyRuntime.interpreters.next_id < 0) {
        /* overflow or Py_Initialize() not called! */
        PyErr_SetString(PyExc_RuntimeError,
                        "failed to get an interpreter ID");
        PyMem_RawFree(interp);
        interp = NULL;
    } else {
        interp->id = _PyRuntime.interpreters.next_id;//将当前is对象插入到rt对象的is对象链表中
        _PyRuntime.interpreters.next_id += 1;
        interp->next = _PyRuntime.interpreters.head;
        if (_PyRuntime.interpreters.main == NULL) {
            _PyRuntime.interpreters.main = interp;//设置runtime对象的主is对象为当前的is对象
        }
        _PyRuntime.interpreters.head = interp;//将is对象链接到runtime对象上
    }
    HEAD_UNLOCK();

    if (interp == NULL) {
        return NULL;
    }

    interp->tstate_next_unique_id = 0;

    return interp;
}
~~~

ts对象的创建

~~~C
static PyThreadState *
new_threadstate(PyInterpreterState *interp, int init)
{
    PyThreadState *tstate = (PyThreadState *)PyMem_RawMalloc(sizeof(PyThreadState));

    if (_PyThreadState_GetFrame == NULL)
        _PyThreadState_GetFrame = threadstate_getframe;//threadstate_getframe返回ts对象的frame域

    if (tstate != NULL) {
        tstate->interp = interp;

        /*...*/
        //ts对象各个域的初始化
        tstate->id = ++interp->tstate_next_unique_id;

        if (init)
            _PyThreadState_Init(tstate);

        HEAD_LOCK();
        tstate->prev = NULL;
        tstate->next = interp->tstate_head;//将当前的ts对象插入到is对象的ts链表中
        if (tstate->next)
            tstate->next->prev = tstate;
        interp->tstate_head = tstate;
        HEAD_UNLOCK();
    }

    return tstate;
}
~~~

现在，我们可以建立rt对象、is以及ts对象这三者之间的联系了。

![image-20210309123520404](Python运行环境初始化.assets/image-20210309123520404.png)

在创建完成is以及ts对象之后，虚拟机还对这些对象进行了配置工作，之后虚拟机调用了pycore_init_types进行了类型系统的初始化。类型系统的初始化是一套相当复杂的动作，我们在《Python虚拟机的类机制》已经进行了详细的分析。

## \__builtin__的创建过程

在pyhton中，如果我们在交互模式下输入“dir()”，那么屏幕上会输出一个list的内容。我们已经知道，python要执行dir这个函数，必定是在某个名字空间中找到了符号“dir”所对应的对象，并且调用了这个对象。在python启动后，我们虽然没有进行任何操作，当时python已经在某个名字空间中创建了符号“dir”及其对应的对象。这个名字空间中的符号和对象来自于系统module，而系统module正是在Py_InitializeEx中创建的。我们首先来看\__builtin__的创建过程。

~~~C
static _PyInitError
pycore_init_builtins(PyInterpreterState *interp)
{
    PyObject *bimod = _PyBuiltin_Init();//在这里进行了builtin模块的初始化
    if (bimod == NULL) {
        return _Py_INIT_ERR("can't initialize builtins modules");
    }
    _PyImport_FixupBuiltin(bimod, "builtins", interp->modules);

    interp->builtins = PyModule_GetDict(bimod);//获取builtin模块的md_dict指针
    if (interp->builtins == NULL) {
        return _Py_INIT_ERR("can't initialize builtins dict");
    }
    Py_INCREF(interp->builtins);

    _PyInitError err = _PyBuiltins_AddExceptions(bimod);
    if (_Py_INIT_FAILED(err)) {
        return err;
    }
    return _Py_INIT_OK();
}
~~~

builtin模块是在_PyBuiltin_Init中初始化的，我们来看一下具体实现。

~~~C
PyObject *
_PyBuiltin_Init(void)
{
    PyObject *mod, *dict, *debug;

    const _PyCoreConfig *config = &_PyInterpreterState_GET_UNSAFE()->core_config;

    if (PyType_Ready(&PyFilter_Type) < 0 ||
        PyType_Ready(&PyMap_Type) < 0 ||
        PyType_Ready(&PyZip_Type) < 0)
        return NULL;

    mod = _PyModule_CreateInitialized(&builtinsmodule, PYTHON_API_VERSION);//初始化builtin模块
    if (mod == NULL)
        return NULL;
    dict = PyModule_GetDict(mod);//将所有pyhton内建类型加入builtin模块中

#ifdef Py_TRACE_REFS
    /* "builtins" exposes a number of statically allocated objects
     * that, before this code was added in 2.3, never showed up in
     * the list of "all objects" maintained by Py_TRACE_REFS.  As a
     * result, programs leaking references to None and False (etc)
     * couldn't be diagnosed by examining sys.getobjects(0).
     */
#define ADD_TO_ALL(OBJECT) _Py_AddToAllObjects((PyObject *)(OBJECT), 0)
#else
#define ADD_TO_ALL(OBJECT) (void)0
#endif

#define SETBUILTIN(NAME, OBJECT) \
    if (PyDict_SetItemString(dict, NAME, (PyObject *)OBJECT) < 0)       \
        return NULL;                                                    \
    ADD_TO_ALL(OBJECT)
	//将所有内建类型放入builtin模块中
    SETBUILTIN("None",                  Py_None);
    /**/
    SETBUILTIN("zip",                   &PyZip_Type);
    debug = PyBool_FromLong(config->optimization_level == 0);
    if (PyDict_SetItemString(dict, "__debug__", debug) < 0) {
        Py_DECREF(debug);
        return NULL;
    }
    Py_DECREF(debug);

    return mod;
#undef ADD_TO_ALL
#undef SETBUILTIN
}
~~~

可以看到，_PyBuiltin_Init是通过两个步骤来完成对builtin的设置：

1. 创建PyModuleObject对象，在python中，模块正是通过这个对象实现的
2. 设置module，将python中所有的类型对象全部放到新创建的builtin对象中

### 创建module对象

在函数_PyModule_CreateInitialized中实现了对PyModuleObject的初始化，该函数接受两个参数：

1. 指向struct PyModuleDef结构体的指针，这个结构体描述了一个python模块的详细信息
2. int类型的参数module_api_version，这是python内部使用的version值，用于比较

首先来看一下PyModuleDef的定义以及builtinsmodule的初始化过程

~~~C
typedef struct PyModuleDef{
  PyModuleDef_Base m_base;
  const char* m_name; //名字
  const char* m_doc;  //文档
  Py_ssize_t m_size;
  PyMethodDef *m_methods; //模块中的函数集合
  struct PyModuleDef_Slot* m_slots;
  traverseproc m_traverse;
  inquiry m_clear;
  freefunc m_free;
} PyModuleDef;
//描述builtin模块的ModuleDef
static struct PyModuleDef builtinsmodule = {
    PyModuleDef_HEAD_INIT,
    "builtins", //名字
    builtin_doc, //模块文档
    -1, /* multiple "initialization" just copies the module dict. */
    builtin_methods, //builtin中的函数集合
    NULL,
    NULL,
    NULL,
    NULL
};
~~~

在_PyModule_CreateInitialized中完成了builtin模块的主要创建工作

~~~C
PyObject *
_PyModule_CreateInitialized(struct PyModuleDef* module, int module_api_version)
{
    const char* name;
    PyModuleObject *m;

    if (!PyModuleDef_Init(module))//检查ModuleDef模块
        return NULL;
    name = module->m_name;
    if (!check_api_version(name, module_api_version)) {
        return NULL;
    }
    /*...*/
    if ((m = (PyModuleObject*)PyModule_New(name)) == NULL)//创建Module对象
        return NULL;

    /*...*/

    if (module->m_methods != NULL) {//将builtin_methods中的方法添加到builtin模块中
        if (PyModule_AddFunctions((PyObject *) m, module->m_methods) != 0) {
            Py_DECREF(m);
            return NULL;
        }
    }
    if (module->m_doc != NULL) {//设置模块doc
        if (PyModule_SetDocString((PyObject *) m, module->m_doc) != 0) {
            Py_DECREF(m);
            return NULL;
        }
    }
    m->md_def = module;
    return (PyObject*)m;
}
~~~

在上面的过程中，对builtin_methods中的方法的会被添加到新创建的builtin模块中，我们来看一下这个builtin_methods。

~~~C
typedef PyObject *(*PyCFunction)(PyObject *, PyObject *);

struct PyMethodDef {
    const char  *ml_name;   /* The name of the built-in function/method */
    PyCFunction ml_meth;    /* The C function that implements it */
    int         ml_flags;   /* Combination of METH_xxx flags, which mostly
                               describe the args expected by the C func */
    const char  *ml_doc;    /* The __doc__ attribute, or NULL */
};
typedef struct PyMethodDef PyMethodDef;

static PyMethodDef builtin_methods[] = {
    {"__build_class__", (PyCFunction)(void(*)(void))builtin___build_class__,
     METH_FASTCALL | METH_KEYWORDS, build_class_doc},
    {"__import__",      (PyCFunction)(void(*)(void))builtin___import__, METH_VARARGS | METH_KEYWORDS, import_doc},
    BUILTIN_ABS_METHODDEF
    /*...*/
    {"iter",            (PyCFunction)(void(*)(void))builtin_iter,       METH_FASTCALL, iter_doc},
    BUILTIN_LEN_METHODDEF
    BUILTIN_LOCALS_METHODDEF
    {"max",             (PyCFunction)(void(*)(void))builtin_max,        METH_VARARGS | METH_KEYWORDS, max_doc},
    /*...*/
    {"vars",            builtin_vars,       METH_VARARGS, vars_doc},
    {NULL,              NULL},
};
~~~

在这里我们发现，dir、len、print等等python内置的函数都是在这里被塞到builtin中的。我们以len为例来看一下PyMethodDef的结构。

![image-20210326145637335](Python运行环境初始化.assets/image-20210326145637335.png)

对于builtin_methods中的每一个PyMethodDef结构，_PyModule_CreateInitialized都会基于它创建一个PyCFunctionObject对象。

~~~C
typedef struct {
    PyObject_HEAD
    PyMethodDef *m_ml; /* Description of the C function to call */
    PyObject    *m_self; /* Passed as 'self' arg to the C func, can be NULL */
    PyObject    *m_module; /* The __module__ attribute, can be anything */
    PyObject    *m_weakreflist; /* List of weak references */
} PyCFunctionObject;

PyObject *					//PyMethodDef 方法所属的模块   模块的名字
PyCFunction_NewEx(PyMethodDef *ml, PyObject *self, PyObject *module)
{
    PyCFunctionObject *op;
    op = free_list;//显然，CFunction也使用了缓冲池机制
    if (op != NULL) {
        free_list = (PyCFunctionObject *)(op->m_self);
        (void)PyObject_INIT(op, &PyCFunction_Type);
        numfree--;
    }
    else {
        op = PyObject_GC_New(PyCFunctionObject, &PyCFunction_Type);
        if (op == NULL)
            return NULL;
    }
    op->m_weakreflist = NULL;
    op->m_ml = ml;
    Py_XINCREF(self);
    op->m_self = self;//所属的模块
    Py_XINCREF(module);
    op->m_module = module;//所属的模块的名字
    _PyObject_GC_TRACK(op);
    return (PyObject *)op;
}
~~~

执行到这里，创建builtin模块的第一部算是基本完成了。接下来，再把python中的内建类型放到builtin中，builtin模块的创建就算基本完成了。结合pycore_init_builtins中对interp->builtins指针的设置，我们可以大致勾勒出建立完成的builtin模块的示意图。

![image-20210329083402055](Python运行环境初始化.assets/image-20210329083402055.png)